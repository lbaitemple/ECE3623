---
title: Course Modules
layout: default
nav_order: 3
has_children: true
---

# Course Modules

ECE 3623 Embedded Systems is organized into **15 comprehensive modules** across **5 progressions**, guiding you from foundational concepts to advanced hardware acceleration.

## Module Structure

Each module includes:
- **Learning Objectives**: Clear goals for what you'll learn
- **Lecture Materials**: Slides and notes
- **Reading Assignments**: Textbook chapters and supplementary materials
- **Hands-on Examples**: Code samples and demonstrations
- **Practice Problems**: Self-assessment questions

---

## Five Progressions Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    COURSE PROGRESSION MAP                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PROGRESSION 1          PROGRESSION 2          PROGRESSION 3    │
│  ┌────────────┐        ┌────────────┐        ┌────────────┐     │
│  │ Foundations│───────►│Synchroniza-│───────►│ Timing &   │     │
│  │ Modules 1-3│        │   tion     │        │ Peripherals│     │
│  │            │        │ Modules 4-7│        │ Modules 8-9│     │
│  └────────────┘        └────────────┘        └────────────┘     │
│                                                     │            │
│                                                     ▼            │
│                        PROGRESSION 5          PROGRESSION 4     │
│                       ┌────────────┐        ┌────────────┐      │
│                       │  Hardware  │◄───────│   Signal   │      │
│                       │Acceleration│        │ Processing │      │
│                       │Modules13-15│        │Modules10-12│      │
│                       └────────────┘        └────────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Progression Summary

| Progression | Title | Modules | Focus |
|-------------|-------|---------|-------|
| **1** | [Foundations](progression1/) | 1-3 | Zynq architecture, RTOS concepts, task management |
| **2** | [Synchronization](progression2/) | 4-7 | Mutexes, priority inversion, interrupts, queues |
| **3** | [Timing & Peripherals](progression3/) | 8-9 | Software timers, watchdog, SPI/ADC |
| **4** | [Signal Processing](progression4/) | 10-12 | ADC theory, DAC waveforms, digital filters |
| **5** | [Hardware Acceleration](progression5/) | 13-15 | HLS, optimization directives, Simulink HDL Coder |

---

## Complete Module List

### Progression 1: Foundations (Weeks 1-3)
| Module | Topic |
|--------|-------|
| [Module 1](module01/) | Introduction to Embedded SoC and Zynq Architecture |
| [Module 2](module02/) | Real-Time Operating System (RTOS) Fundamentals |
| [Module 3](module03/) | FreeRTOS Task Management |

### Progression 2: Synchronization (Weeks 4-7)
| Module | Topic |
|--------|-------|
| [Module 4](module04/) | Mutual Exclusion and Resource Sharing |
| [Module 5](module05/) | Priority Inversion and Inheritance |
| [Module 6](module06/) | Hardware Interrupts and GIC |
| [Module 7](module07/) | Queues and Inter-Task Communication |

### Progression 3: Timing & Peripherals (Weeks 8-9)
| Module | Topic |
|--------|-------|
| [Module 8](module08/) | Software Timers and Watchdog |
| [Module 9](module09/) | ADC and SPI Communication |

### Progression 4: Signal Processing (Weeks 10-12)
| Module | Topic |
|--------|-------|
| [Module 10](module10/) | Analog-to-Digital Conversion |
| [Module 11](module11/) | Digital-to-Analog Conversion |
| [Module 12](module12/) | Digital Signal Processing |

### Progression 5: Hardware Acceleration (Weeks 13-15)
| Module | Topic |
|--------|-------|
| [Module 13](module13/) | High-Level Synthesis |
| [Module 14](module14/) | HLS Directives and Optimization |
| [Module 15](module15/) | Model-Based Design with Simulink |

---

## Recommended Study Path

1. **Complete each progression in order** - Concepts build upon previous progressions
2. **Do the readings before lecture** - Come prepared with questions
3. **Complete hands-on exercises** - Practice reinforces learning
4. **Start labs early** - Allow time for debugging and office hours
5. **Review before exams** - Use module summaries and practice problems
