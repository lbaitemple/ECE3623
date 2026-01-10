---
title: "Module 1: Introduction to Embedded SoC and Zynq Architecture"
layout: default
parent: "Progression 1: Foundations"
grand_parent: Course Modules
nav_order: 1
---

# Module 1: Introduction to Embedded SoC and Zynq Architecture
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the architecture of the Zynq-7000 All Programmable SoC
2. Describe the integration between the Processing System (PS) and Programmable Logic (PL)
3. Identify the components of the Application Processor Unit (APU)
4. Distinguish between soft and hard processor cores
5. Understand the four-layer architecture of the Zynq platform

---

## Overview

The Zynq-7000 All Programmable SoC (AP SoC) represents a paradigm shift in embedded system design by tightly integrating a powerful ARM-based Processing System with flexible FPGA Programmable Logic on a single chip. This architecture enables designers to leverage the best of both worlds: the ease of software development with the performance of custom hardware.

![Zynq Architecture Overview]({{ site.baseurl }}/docs/imgs/pdf_images/embedded_system/img-000.png)

---

## The Zynq-7000 AP SoC Architecture

### Processing System (PS) and Programmable Logic (PL)

The Zynq architecture fundamentally consists of two main domains:

| Domain | Description | Key Features |
|--------|-------------|--------------|
| **Processing System (PS)** | Hard silicon ARM processor subsystem | Always present, boots first, controls PL |
| **Programmable Logic (PL)** | FPGA fabric for custom hardware | Optional, configured by PS, Artix-7/Kintex-7 class |

```
┌─────────────────────────────────────────────────────────────────┐
│                      Zynq-7000 AP SoC                           │
├───────────────────────────┬─────────────────────────────────────┤
│   Processing System (PS)  │     Programmable Logic (PL)         │
│                           │                                     │
│  ┌─────────────────────┐  │  ┌─────────────────────────────┐   │
│  │ Application         │  │  │  Custom Hardware Blocks     │   │
│  │ Processor Unit      │  │  │  - Accelerators             │   │
│  │ (APU)               │  │  │  - Custom Peripherals       │   │
│  │ - Dual ARM Cortex-A9│  │  │  - Signal Processing        │   │
│  └─────────────────────┘  │  └─────────────────────────────┘   │
│                           │                                     │
│  ┌─────────────────────┐  │  ┌─────────────────────────────┐   │
│  │ Memory Interfaces   │◄─┼─►│  AXI Interconnect           │   │
│  │ - DDR Controller    │  │  │  - GP Ports (32-bit)        │   │
│  │ - On-chip Memory    │  │  │  - HP Ports (64-bit)        │   │
│  └─────────────────────┘  │  │  - ACP Port                 │   │
│                           │  └─────────────────────────────┘   │
│  ┌─────────────────────┐  │                                     │
│  │ I/O Peripherals     │  │  ┌─────────────────────────────┐   │
│  │ - UART, SPI, I2C    │  │  │  FPGA Resources             │   │
│  │ - USB, Ethernet     │  │  │  - CLBs, BRAM, DSP          │   │
│  │ - GPIO, Timers      │  │  │  - I/O Banks                │   │
│  └─────────────────────┘  │  └─────────────────────────────┘   │
└───────────────────────────┴─────────────────────────────────────┘
```

### Key Integration Points

The PS and PL communicate through multiple high-bandwidth interfaces:

- **AXI GP Ports**: General-purpose 32-bit interfaces for control and status
- **AXI HP Ports**: High-performance 64-bit interfaces for data streaming
- **AXI ACP Port**: Accelerator Coherency Port for cache-coherent access
- **EMIO**: Extended Multiplexed I/O for routing PS peripherals through PL
- **Interrupts**: PL can generate interrupts to PS (IRQ_F2P)

---

## Application Processor Unit (APU)

The APU is the heart of the Processing System, containing the ARM processing cores and associated hardware.

### APU Components

| Component | Description |
|-----------|-------------|
| **Dual ARM Cortex-A9 MPCore** | Up to 1 GHz, superscalar, out-of-order execution |
| **NEON Coprocessor** | 128-bit SIMD for media processing |
| **FPU** | Floating-point unit (VFPv3) |
| **MMU** | Memory Management Unit for virtual memory |
| **L1 Cache** | 32KB instruction + 32KB data per core |
| **L2 Cache** | 512KB shared between cores |
| **SCU** | Snoop Control Unit for cache coherency |

### ARM Cortex-A9 Features

```
┌─────────────────────────────────────────────┐
│           ARM Cortex-A9 MPCore              │
├─────────────────────┬───────────────────────┤
│       CPU 0         │        CPU 1          │
│  ┌──────┬──────┐    │   ┌──────┬──────┐    │
│  │32KB  │32KB  │    │   │32KB  │32KB  │    │
│  │I-$   │D-$   │    │   │I-$   │D-$   │    │
│  └──────┴──────┘    │   └──────┴──────┘    │
│  ┌──────────────┐   │   ┌──────────────┐   │
│  │ NEON + FPU   │   │   │ NEON + FPU   │   │
│  └──────────────┘   │   └──────────────┘   │
├─────────────────────┴───────────────────────┤
│        Snoop Control Unit (SCU)             │
├─────────────────────────────────────────────┤
│          512KB L2 Cache                     │
└─────────────────────────────────────────────┘
```

---

## Hard vs Soft Processor Cores

Understanding the distinction between hard and soft cores is essential for Zynq development.

### Hard Processor Core

A **hard core** is implemented in dedicated silicon (fixed transistors):

| Advantages | Disadvantages |
|------------|---------------|
| Higher clock speeds (up to 1 GHz) | Fixed functionality |
| Lower power consumption | Cannot be modified |
| Smaller silicon area | Limited to what's on chip |
| Deterministic performance | |

**Example**: The ARM Cortex-A9 in Zynq PS is a hard core.

### Soft Processor Core

A **soft core** is implemented using FPGA programmable logic:

| Advantages | Disadvantages |
|------------|---------------|
| Fully customizable | Lower clock speeds (~100-300 MHz) |
| Can instantiate multiple cores | Higher power consumption |
| Can be modified/updated | Uses FPGA resources |
| Application-specific optimization | |

**Examples**: MicroBlaze, PicoBlaze, RISC-V soft cores.

### When to Use Each

| Use Case | Recommended |
|----------|-------------|
| Complex OS (Linux, FreeRTOS) | Hard core (PS) |
| High-speed computation | Hard core (PS) |
| Simple state machines | Soft core or pure logic |
| Multiple specialized controllers | Soft cores in PL |
| Custom instruction sets | Soft core |

---

## Four-Layer Zynq Architecture

The Zynq platform can be understood through a four-layer model:

### Layer 1: Application Layer
- User applications and algorithms
- Operating system (FreeRTOS, Linux)
- Device drivers and middleware

### Layer 2: OS/Abstraction Layer
- Hardware abstraction layer (HAL)
- Board support packages (BSP)
- Xilinx standalone or FreeRTOS

### Layer 3: Hardware Platform Layer
- PS configuration (peripherals, clocks)
- PL custom IP blocks
- Interconnect configuration

### Layer 4: Physical Layer
- Silicon (PS hard blocks)
- FPGA fabric (PL)
- Package and I/O

```
┌─────────────────────────────────────┐
│     Layer 4: Applications           │
│     (User code, Algorithms)         │
├─────────────────────────────────────┤
│     Layer 3: OS & Abstraction       │
│     (FreeRTOS, Drivers, BSP)        │
├─────────────────────────────────────┤
│     Layer 2: Hardware Platform      │
│     (PS Config, Custom IP, AXI)     │
├─────────────────────────────────────┤
│     Layer 1: Physical               │
│     (Silicon, FPGA Fabric, I/O)     │
└─────────────────────────────────────┘
```

---

## Development Environment

### Vivado Design Suite

Vivado handles all hardware design aspects:

1. **IP Integrator**: Block-based system design
2. **Synthesis**: Converts HDL to netlist
3. **Implementation**: Place and route
4. **Bitstream Generation**: Creates PL configuration file

### Vitis Unified IDE

Vitis handles software development:

1. **Platform Creation**: From Vivado hardware export
2. **Application Development**: C/C++ for ARM
3. **Debugging**: JTAG-based debug
4. **Profiling**: Performance analysis

### Design Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Vivado     │────►│  Export HW   │────►│    Vitis     │
│  Block Design│     │   (.xsa)     │     │  Application │
└──────────────┘     └──────────────┘     └──────────────┘
       │                                         │
       ▼                                         ▼
┌──────────────┐                         ┌──────────────┐
│  Bitstream   │                         │   .elf       │
│   (.bit)     │                         │  (firmware)  │
└──────────────┘                         └──────────────┘
       │                                         │
       └───────────────┬─────────────────────────┘
                       ▼
              ┌──────────────┐
              │  Program     │
              │  Zynq Board  │
              └──────────────┘
```

---

## Hands-On: Exploring Zynq PS Configuration

### Step 1: Create Vivado Project
1. Launch Vivado
2. Create New Project → RTL Project
3. Select your board (Zybo Z7-10 or Z7-20)

### Step 2: Examine PS Configuration
1. Create Block Design
2. Add ZYNQ7 Processing System IP
3. Double-click to open configuration
4. Explore:
   - **PS-PL Configuration**: AXI interfaces
   - **Peripheral I/O Pins**: MIO allocation
   - **Clock Configuration**: PS and PL clocks
   - **DDR Configuration**: Memory settings

### Step 3: Key Observations
- Which peripherals are enabled by default?
- How many AXI GP/HP ports are available?
- What clock frequencies are configured?

---

## Lecture Materials

- [Embedded System Introduction]({{ site.baseurl }}/ece3623/Embedded_System.pdf)
- [Week 1 Slides]({{ site.baseurl }}/ece3623/Week%201.pdf)
- [Week 1 Recap]({{ site.baseurl }}/ece3623/Week%201-R.pdf)
- [The Zynq Book Tutorials]({{ site.baseurl }}/ece3623/The_Zynq_Book_Tutorials.pdf)

---

## Reading Assignments

1. *The Zynq Book*, Chapter 1: Introduction to Zynq
2. *The Zynq Book*, Chapter 2: The Zynq Architecture
3. Zynq-7000 Technical Reference Manual, Chapter 1: Introduction

---

## Practice Questions

1. What are the two main domains of the Zynq-7000 architecture?
2. List three components of the Application Processor Unit (APU).
3. Explain the difference between a hard processor core and a soft processor core.
4. What is the purpose of the Snoop Control Unit (SCU)?
5. Describe the four layers of the Zynq architecture model.
6. Which boots first in Zynq: the PS or the PL? Why?
7. What are the three types of AXI interfaces between PS and PL?

---

## Summary

This module introduced the Zynq-7000 AP SoC architecture, emphasizing the tight integration between the Processing System (ARM cores) and Programmable Logic (FPGA fabric). Understanding the APU composition, the distinction between hard and soft cores, and the four-layer architecture provides the foundation for developing sophisticated embedded systems that leverage both software flexibility and hardware performance.

---

## Next Module

[Module 2: Real-Time Operating System (RTOS) Fundamentals →](../module02/)
