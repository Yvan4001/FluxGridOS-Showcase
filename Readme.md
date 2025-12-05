# FluxGridOS

A custom operating system kernel optimized for gaming, written in Rust.

## Overview

FluxGridOS is a from-scratch operating system kernel designed specifically for gaming environments. Built with Rust's safety guarantees and zero-cost abstractions, this project aims to create a minimal, high-performance OS that dedicates maximum resources to gaming applications while providing necessary system services.

## Features

**Custom Assembly Bootloader**: Hand-written in ASM to handle specific hardware initialization and memory mapping where standard loaders failed. 
- **Bare-metal Rust Kernel**: High-level safety with low-level control.
- **Memory Optimized**: Custom memory management (`kernel/memory/`) tuned for gaming workloads.
- **CPU & Interrupt Management**: Dedicated modules for CPU features (`kernel/cpu/`) and interrupt handling (`kernel/interrupts/`).
- **Modular Drivers (`kernel/drivers/`)**: Including GPU-focused drivers (NVIDIA support), input, sound, and storage.
- **Low-latency Audio**: Custom sound subsystem for minimal audio latency.
- **Configuration System (`config/`)**: Binary configuration for persistent settings and gaming profiles.
- **GUI Layer (`gui/`)**: With custom widgets (`gui/widgets/`) and asset management (`gui/assets/`).
- **Gaming Profiles**: Support for different optimization profiles.

## Conceptual Architecture
```bash
┌────────────────────────────────────────────────┐
│                  Applications                   │
└───────────────────────┬────────────────────────┘
│
┌───────────────────────┴────────────────────────┐
│               GUI Layer (gui/)               │
│     (Windowing, Renderer, Widgets, Assets)     │
└───────────────────────┬────────────────────────┘
│
┌───────────────────────┴────────────────────────┐
│            FluxGrid Kernel (kernel/)         │
├────────────────┬──────────────┬────────────────┤
│ Memory Manager │  Scheduler   │ Device Drivers  │
│ (kernel/memory)|(Core Kernel) │ (kernel/drivers)|
├────────────────┼──────────────┼────────────────┤
│ CPU Management │ Interrupts   │ Core Services  │
│ (kernel/cpu) │ (kernel/interrupts)|              │
├────────────────┴──────────────┴────────────────┤
│              Hardware Abstraction Layer         │
└────────────────────────┬─────────────────────┬─┘
│                     │
┌────────────────────────┴─┐   ┌───────────────┴─┐
│     System Hardware      │   │       BIOS       │
└──────────────────────────┘   └─────────────────┘
```
## Detailed Architecture

FluxGridOS follows a modular design with several key components organized as follows:

### 0. Bootloader (`bootloader/`)

The custom assembly bootloader is the first code that runs and prepares the system for the Rust kernel.

-   **Architecture:** Pure x86_64 Assembly (NASM syntax)
-   **Standards Compliance:**
    -   **Multiboot 2 Header:** Primary boot method for GRUB and ISO booting
        -   Magic: `0xE85250D6`
        -   Framebuffer Tag: Requests 1920×1080×32bpp linear framebuffer
        -   Ensures graphics mode (not VGA text mode) for GUI/gaming
    -   **Multiboot 1 Header:** Fallback for QEMU `-kernel` direct boot
        -   Magic: `0x1BADB002`
        -   Video Mode Info: Specifies linear graphics mode, resolution, and color depth
        -   Flags: `0x07` (Align modules + Memory Info + Video Mode)

-   **Boot Sequence:**
    1.  **Entry (32-bit Protected Mode):** GRUB/bootloader loads kernel and jumps to `start`
    2.  **Page Table Setup (`setup_page_tables`):**
        -   Creates 4-level paging hierarchy (P4→P3→P2→Physical)
        -   Recursive mapping: P4[511] → P4 (allows runtime page table modification)
        -   Identity mapping: Virtual 0x0 → Physical 0x0 (first 4GB)
        -   Higher-half mapping: Virtual 0xFFFF_8000_0000_0000 → Physical 0x0 (kernel space)
        -   Uses 2MB huge pages for efficiency (512 entries × 4 tables = 4GB coverage)
    3.  **Paging Enablement (`enable_paging`):**
        -   Loads P4 table address into CR3
        -   Enables PAE (Physical Address Extension) in CR4
        -   Sets LME (Long Mode Enable) bit in EFER MSR
        -   Activates paging via CR0.PG bit
    4.  **SSE Initialization (`enable_sse`):**
        -   Clears CR0.EM (no FPU emulation)
        -   Sets CR0.MP (monitor coprocessor)
        -   Enables OSFXSR and OSXMMEXCPT in CR4 for SSE support
    5.  **GDT Loading:** Loads 64-bit Global Descriptor Table with code/data segments
    6.  **Mode Switch:** Far jump to 64-bit code segment (`long_mode_start`)
    7.  **Segment Setup (64-bit Mode):** Initializes all segment registers (SS, DS, ES, FS, GS)
    8.  **Kernel Entry:** Calls Rust `_start` function with Multiboot info preserved

-   **Memory Layout:**
    -   `.multiboot_header` section: Placed at start of binary for bootloader detection
    -   `.text` section: Contains all boot code (32-bit and 64-bit)
    -   `.bss` section: Uninitialized data (page tables, stack)
        -   5 page tables: 5 × 4KB = 20KB
        -   Stack: 64KB (grows downward from `stack_top`)
    -   `.rodata` section: GDT structure (read-only)

-   **Key Features:**
    -   **Zero dependencies:** No reliance on external boot libraries
    -   **Framebuffer ready:** Graphics mode configured before kernel starts
    -   **Large address space:** 4GB initially mapped, expandable by kernel
    -   **Security:** NX bit support via proper page table flags
    -   **Debug friendly:** Preserves Multiboot info for kernel introspection

### 1. Kernel (`kernel/`)

The core of the operating system, responsible for all low-level operations and resource management.

-   **Boot Process (`kernel/boot/`):**
    - **Multiboot2 Integration (`boot/info.rs`):** Parses Multiboot2 boot information from GRUB
    - **Boot Sequencing (`boot/setup.rs`):** Orchestrates kernel initialization phases
    - **Memory Map Parsing:** Extracts and validates physical memory regions from bootloader
    - **Framebuffer Detection:** Retrieves graphics mode information (address, resolution, pitch, bpp)
    - **Early Logging:** Sets up serial output before full driver initialization
    
-   **CPU Management (`kernel/cpu/`):**
    - **GDT Management (`gdt.rs`):** Global Descriptor Table with 64-bit code/data segments
    - **TSS Setup:** Task State Segment for interrupt stack switching
    - **CPU Feature Detection:** CPUID-based capability enumeration
    - **Exception Handling:** Double fault, page fault, and other CPU exceptions
    
-   **Interrupt Handling (`kernel/interrupts/`):**
    - **IDT Setup (`idt.rs`):** Interrupt Descriptor Table with 256 entries
    - **Hardware IRQs:** Timer (IRQ0), Keyboard (IRQ1), Mouse (IRQ12)
    - **PIC Configuration (`pic.rs`):** 8259 Programmable Interrupt Controller management
    - **Exception Handlers:** CPU fault handlers (page fault, general protection, etc.)
    
-   **Memory Subsystem (`kernel/memory/`):**
    - **Physical Allocator (`physical.rs`):** Frame-based allocation with bitmap tracking
    - **Virtual Memory (`virtual.rs`):** 4-level page table management (P4→P3→P2→P1)
    - **Heap Allocator (`allocator.rs`):** Linked-list allocator for kernel heap (4MB default)
    - **Memory Manager (`memory_manager.rs`):** Unified interface coordinating physical/virtual memory
    - **Page Mapping:** PRESENT, WRITABLE, NO_CACHE flags for framebuffer and MMIO regions
    
-   **Device Drivers (`kernel/drivers/`):**
    - **Video Subsystem (`drivers/video/`):**
        - **VGA Driver (`vga.rs`):** Text mode console for early boot (80×25 characters)
        - **VESA Driver (`vesa.rs`):** Generic framebuffer driver using Multiboot-provided framebuffer
        - **HDMI Driver (`hdmi.rs`):** HDMI output with EDID reading, resolution detection
        - **DisplayPort (`displayport.rs`):** DisplayPort protocol support
        - **Framebuffer Console (`framebuffer_console.rs`):** Software text rendering on framebuffer
        
    - **GPU Subsystem (`drivers/gpu/`):**
        - **GPU Detection (`gpu/detection.rs`):** PCI enumeration for Intel/AMD/NVIDIA GPUs
        - **VESA Fallback:** Software rendering when no hardware acceleration available
        - **Vendor Drivers (`gpu/specific/`):** Intel, AMD, NVIDIA-specific initialization
        - **Common Utilities (`gpu/common.rs`):** MMIO access, register operations
        - **I²C/DDC (`gpu/i2c.rs`):** Display communication for EDID and monitor control
        
    - **Input Drivers:**
        - **Keyboard (`keyboard.rs`):** PS/2 keyboard with scancode translation and modifier tracking
        - **Mouse (`mouse.rs`):** PS/2 mouse with packet parsing and position tracking
        - **USB HID (`usb_hid.rs`):** USB keyboard/mouse/gamepad support
        - **Gamepad (`gamepad.rs`):** Game controller abstraction layer
        
    - **Storage Subsystem (`drivers/storage/`, `drivers/ahci/`, `drivers/ata/`):**
        - **AHCI Driver:** SATA controller for modern drives (NCQ support)
        - **ATA/IDE Driver:** Legacy hard drive support
        - **Storage Manager:** Unified block device interface
        
    - **USB Subsystem (`drivers/usb.rs`):**
        - **UHCI Controller:** Universal Host Controller Interface for USB 1.1
        - **Device Enumeration:** Descriptor parsing, address assignment
        - **Control Transfers:** Setup/data/status phases
        - **Interrupt Endpoints:** Periodic polling for HID devices
        - **USB Tablet Support:** Absolute positioning mouse input
        
    - **Filesystem (`drivers/filesystem.rs`):**
        - **Virtual File System (VFS):** Abstract filesystem interface
        - **File/Directory Handles:** Open, read, write, seek operations
        - **Path Resolution:** Hierarchical directory traversal
        
    - **Other Drivers:**
        - **Sound (`sound.rs`):** Audio output framework
        - **Network (`network.rs`):** Network stack initialization
        - **Timer (`timer.rs`):** Programmable Interval Timer (PIT) for system tick
        - **Power (`power.rs`):** ACPI-based power management
        - **Display (`display.rs`):** Display detection and configuration
        
-   **Driver Manager (`kernel/drivers/mod.rs`):**
    - **Unified Initialization:** Orchestrates all driver initialization in proper order
    - **Dependency Resolution:** Ensures timer/interrupts before device drivers
    - **Error Handling:** Graceful fallback when optional drivers fail
    - **Global Access:** Static mutex-protected driver manager instance
    
-   **Core Kernel Services:**
    - **Serial Logging (`logger.rs`):** COM1 serial port output at 115200 baud
    - **Panic Handler:** Double fault detection with register dumps
    - **Synchronization:** Spinlocks (via `spin` crate) for SMP-safe data structures

### 2. Configuration System (`config/`)

Manages persistent system settings and user preferences.

-   Uses `bincode` for serialization/deserialization of `SystemConfig` structures.
-   Loads configuration at boot and saves changes as needed.
-   Provides default settings if a configuration file is missing or corrupt.
-   Supports "Gaming Profiles" which can apply sets of pre-defined optimizations (e.g., CPU affinity, power settings, driver parameters).

### 3. GUI Layer (`gui/`)

Provides the graphical user interface and rendering capabilities.

-   **Window Management:** Handles window creation, placement, and events.
-   **Renderer (`gui/renderer.rs`):** Abstracts graphics drawing operations, potentially using GPU acceleration via `kernel/drivers/gpu`.
-   **Widgets (`gui/widgets/`):** A library of UI elements (buttons, sliders, text boxes, etc.).
-   **Assets (`gui/assets/`):** Management for fonts, themes, icons, and other UI resources.

### 4. Applications

User-level applications, primarily games, that run on top of the OS and sofware layer.

## Download & Installation

Since FluxGridOS is currently closed-source/proprietary, you cannot build it from source without a specific license. However, you can try the latest builds!

### How to run FluxGridOS:

1.  **Download the ISO:** Go to the [Releases Page](../../releases) and download the latest `FluxGrid.iso`.
2.  **Emulation (QEMU):**
    ```sh
    qemu-system-x86_64 -cdrom FluxGrid.iso -m 2G
    ```
3.  **Real Hardware:** Flash the ISO to a USB drive (using Rufus or Etcher) and boot from it (requires UEFI/Legacy support depending on version).

*Note: As this is a Beta OS, running on a Virtual Machine is highly recommended.*

## Current Status & Roadmap

FluxGridOS is in active development.

### Working Components:
* Boot sequence and initial hardware setup.
* Core memory management (physical and virtual memory, heap).
* VGA text mode output.
* Basic keyboard input.
* Initial structures for GPU drivers and configuration system.

### Planned / In Progress:
* **Full GPU Acceleration:** Complete NVIDIA driver, 2D/3D rendering pipeline.
* **Advanced Scheduler:** Game-optimized, low-latency scheduler.
* **Sound System:** Robust, low-latency audio output and input.
* **Storage Subsystem:** Filesystem support and block device drivers.
* **Networking Stack:** For online gaming and system updates (if applicable).
* **Comprehensive GUI:** Rich widget set, window manager, theming.
* **Gaming Profiles:** Implementation of profile switching and settings.

## Feedback & Issues

While the source code is proprietary, community feedback is invaluable!
If you encounter a bug or have a feature request for the gaming optimization layer:

1.  Open an **Issue** in this repository.
2.  Describe the bug or suggestion clearly.
3.  If reporting a crash, please attach a screenshot or the serial logs if available.

## License

This project is Source Available / Proprietary. You can view and study the code for educational purposes, but commercial use, modification, and redistribution are restricted. See the [LICENSE](LICENSE) file for details.
