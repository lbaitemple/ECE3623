---
title: Laboratory Exercises
layout: default
nav_order: 4
has_children: true
---

# Laboratory Exercises

ECE 3623 includes **7 hands-on laboratory assignments** that reinforce concepts from the course modules. Labs progress from basic FreeRTOS programming to complete signal processing systems.

---

## Lab Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    LABORATORY PROGRESSION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Lab 1              Lab 2              Lab 3                    │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐              │
│  │   Task   │─────►│Interrupts│─────►│  Queues  │              │
│  │Management│      │   AXI    │      │  Data    │              │
│  └──────────┘      └──────────┘      └──────────┘              │
│       │                                    │                     │
│       │                                    ▼                     │
│       │           Lab 4              Lab 5                      │
│       │          ┌──────────┐      ┌──────────┐                │
│       └─────────►│  Timers  │─────►│   ADC    │                │
│                  │ Watchdog │      │ PmodAD1  │                │
│                  └──────────┘      └──────────┘                │
│                                         │                       │
│                                         ▼                       │
│                       Lab 6              Lab 7                  │
│                      ┌──────────┐      ┌──────────┐            │
│                      │   DAC    │─────►│ Digital  │            │
│                      │ PmodDA2  │      │ Filters  │            │
│                      └──────────┘      └──────────┘            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lab Summary

| Lab | Title | Related Modules | Progression |
|-----|-------|-----------------|-------------|
| [Lab 1](lab1/) | Task Management in FreeRTOS | Modules 2-3 | Foundations |
| [Lab 2](lab2/) | Vivado AXI Interrupt | Modules 6-7 | Synchronization |
| [Lab 3](lab3/) | FreeRTOS Data Queue | Modules 7-8 | Synchronization |
| [Lab 4](lab4/) | Software Timers and Watchdog | Modules 8-9 | Timing & Peripherals |
| [Lab 5](lab5/) | PmodAD1 ADC Interface | Modules 9-10 | Signal Processing |
| [Lab 6](lab6/) | PmodDA2 Function Generator | Module 11 | Signal Processing |
| [Lab 7](lab7/) | Digital Filters Implementation | Module 12 | Signal Processing |

---

## Lab Schedule

| Week | Lab | Due Date (Typical) |
|------|-----|-------------------|
| Week 3-4 | Lab 1: Task Management | End of Week 4 |
| Week 5-6 | Lab 2: AXI Interrupt | End of Week 6 |
| Week 7-8 | Lab 3: Data Queue | End of Week 8 |
| Week 9 | Lab 4: Timers & Watchdog | End of Week 9 |
| Week 10 | Lab 5: PmodAD1 ADC | End of Week 10 |
| Week 11-12 | Lab 6: PmodDA2 DAC | End of Week 12 |
| Week 13-14 | Lab 7: Digital Filters | End of Week 14 |

---

## Lab Requirements

### Hardware
- **Zybo Z7-10 or Z7-20** development board
- **PmodAD1** - 12-bit ADC module (Labs 5, 7)
- **PmodDA2** - 12-bit DAC module (Labs 6, 7)
- USB cables, oscilloscope access

### Software
- **Xilinx Vivado 2020.2** (or later)
- **Xilinx Vitis 2020.2** (or later)
- Serial terminal (PuTTY, Tera Term, or similar)

---

## Lab Report Guidelines

Each lab report should include:

1. **Objective** - What you're trying to accomplish
2. **Procedure** - Steps taken to complete the lab
3. **Results** - Screenshots, measurements, outputs
4. **Code Listings** - Key code sections with explanations
5. **Analysis** - Discussion of results and observations
6. **Conclusion** - Summary and lessons learned

---

## Getting Help

- **Office Hours**: See syllabus for schedule
- **Lab Sessions**: TAs available during scheduled lab times
- **Discussion Board**: Post questions on course forum
- **Debugging Tips**: Check each lab's troubleshooting section
