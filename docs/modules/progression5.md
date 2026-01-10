---
title: "Progression 5: Hardware Acceleration"
layout: default
nav_order: 5
parent: Course Modules
has_children: true
---

# Progression 5: Hardware Acceleration

Learn to accelerate algorithms using FPGA programmable logic through HLS and model-based design.

---

## Overview

This progression covers hardware acceleration techniques for Zynq PL:

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [Module 13](module13/) | High-Level Synthesis | C-to-HDL, scheduling, binding, RTL generation |
| [Module 14](module14/) | HLS Directives | PIPELINE, UNROLL, DATAFLOW, latency vs throughput |
| [Module 15](module15/) | Model-Based Design | Simulink, HDL Coder, IP core generation |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│              HARDWARE ACCELERATION PROGRESSION                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Module 13             Module 14             Module 15          │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐       │
│  │   HLS    │          │   HLS    │          │ Simulink │       │
│  │  Basics  │─────────►│Directives│          │   HDL    │       │
│  │ C to HDL │          │Optimize  │          │  Coder   │       │
│  └──────────┘          └──────────┘          └──────────┘       │
│       │                      │                     │             │
│       │                      │                     │             │
│       └──────────────────────┴─────────────────────┘             │
│                              │                                   │
│                              ▼                                   │
│                    ┌──────────────────┐                         │
│                    │  ZYNQ PL DESIGN  │                         │
│                    │  Custom Hardware │                         │
│                    │   Accelerators   │                         │
│                    └──────────────────┘                         │
│                                                                  │
│  Two Paths to FPGA Hardware:                                    │
│  • C/C++ → Vivado HLS → HDL (Modules 13-14)                    │
│  • Simulink → HDL Coder → HDL (Module 15)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Completion of Progression 4 (Signal Processing)
- Understanding of digital filter algorithms (for acceleration examples)

---

## Key Metrics

**Throughput:**
$$\text{Throughput} = \frac{f_{clk}}{II}$$

Where:
- $f_{clk}$ = clock frequency (Hz)
- $II$ = initiation interval (cycles)

---

## Design Flow Comparison

| Approach | Input | Tool | Best For |
|----------|-------|------|----------|
| Vivado HLS | C/C++ | Vivado | Algorithm engineers |
| HDL Coder | Simulink | MATLAB | Control/DSP engineers |
| Manual RTL | Verilog/VHDL | Vivado | Hardware experts |

---

## Outcomes

After completing this progression, you will be able to:

1. Convert C algorithms to hardware using Vivado HLS
2. Apply optimization directives to improve performance
3. Use Simulink and HDL Coder for model-based FPGA design
4. Integrate custom IP cores with Zynq PS
5. Analyze latency, throughput, and resource tradeoffs
