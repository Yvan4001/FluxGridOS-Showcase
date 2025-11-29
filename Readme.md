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

### 1. Kernel (`kernel/`)

The core of the operating system, responsible for all low-level operations and resource management.

-   **Boot Process:**
    - Custom Bootloader: Written in Assembly. Handles the transition from Real Mode to Long Mode, sets up the initial GDT/IDT, and hands off execution to the Rust kernel (Multiboot2 compliant).
    -   Sets up initial paging, GDT, IDT, and other crucial CPU structures.
    -   Transfers control to the Rust kernel entry point (`_start`).
-   **CPU Management (`kernel/cpu/`):**
    -   Manages CPU-specific features, modes, and states.
    -   Global Descriptor Table (GDT) setup.
    -   Task State Segment (TSS) setup.
    -   May include CPU-specific power management or feature detection.
-   **Interrupt Handling (`kernel/interrupts/`):**
    -   Interrupt Descriptor Table (IDT) setup.
    -   Interrupt Service Routines (ISRs) for hardware and software interrupts.
    * Programmable Interrupt Controller (PIC/APIC) management.
-   **Memory Subsystem (`kernel/memory/`):**
    -   **Physical Memory Manager (`physical.rs`):** Tracks available physical memory, handles frame allocation/deallocation, and manages the memory map.
    -   **Virtual Memory Manager (`virtual.rs`):** Manages page tables, virtual address space allocation, memory protection, and permissions.
    -   **Heap Allocator (`allocator.rs`):** Implements the global allocator trait for dynamic memory allocation within the kernel.
    -   **DMA Manager (`dma.rs`):** Handles Direct Memory Access for devices, managing DMA-safe memory regions and ensuring coherence.
-   **Device Drivers (`kernel/drivers/`):**
    -   Manages interaction with hardware components. This is a modular system allowing for different drivers to be loaded or compiled in.
    -   **GPU Drivers:** Specialized support, initially targeting NVIDIA. Includes VRAM management, command submission, and 2D/3D acceleration interfaces.
    -   **Display Drivers:** Handles video output modes (e.g., VGA text mode for early debug, GOP/DRM for graphical modes, HDMI).
    -   **Input Drivers:** Keyboard, mouse, and potentially gamepad controllers.
    -   **Sound Drivers:** Interacts with sound hardware for audio playback and recording.
    -   **Storage Drivers:** Controllers for storage devices (e.g., AHCI for SATA drives, NVMe).
    -   **Platform Drivers:** ACPI for power management and system configuration, PCI/PCIe bus enumeration.
-   **Scheduler:**
    -   Manages task/thread scheduling. (Details to be expanded, e.g., pre-emptive, priority-based, game-optimized).
-   **Core Kernel Services:**
    -   May include system call interfaces (if user-mode is planned).
    -   Synchronization primitives.
    -   Kernel-internal logging and debugging facilities.

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
    qemu-system-x86_64 -cdrom FluxGrid.iso -m 512M
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
