---
title: Course Modules
layout: default
nav_order: 3
has_children: true
---

# Course Modules

ECE 3623 Embedded Systems is organized into 14 comprehensive modules, progressing from foundational concepts to advanced system integration.

## Module Structure

Each module includes:
- **Learning Objectives**: Clear goals for what you'll learn
- **Lecture Materials**: Slides and notes
- **Reading Assignments**: Textbook chapters and supplementary materials
- **Hands-on Examples**: Code samples and demonstrations
- **Practice Problems**: Self-assessment questions

---

## Module Progression

### Foundations (Modules 1-2)
Build your understanding of embedded systems architecture and real-time computing concepts.

| Module | Topic | Duration |
|--------|-------|----------|
| [Module 1](module01/) | Introduction to Embedded Systems | Week 1 |
| [Module 2](module02/) | Real-Time Systems | Week 2 |

### RTOS Programming (Modules 3-4, 8)
Master FreeRTOS programming for real-time applications.

| Module | Topic | Duration |
|--------|-------|----------|
| [Module 3](module03/) | FreeRTOS Fundamentals | Week 3 |
| [Module 4](module04/) | Task Management & Scheduling | Week 4 |
| [Module 8](module08/) | Inter-Task Communication | Week 7 |

### Timing & Hardware (Modules 5-7)
Understand timer hardware and interrupt-driven programming.

| Module | Topic | Duration |
|--------|-------|----------|
| [Module 5](module05/) | Counter-Timers | Week 5 |
| [Module 6](module06/) | Timer Measurement & Watchdog | Week 6 |
| [Module 7](module07/) | Interrupts & Exception Handling | Week 6-7 |

### Communication & I/O (Modules 9-11)
Interface with external peripherals and analog signals.

| Module | Topic | Duration |
|--------|-------|----------|
| [Module 9](module09/) | Serial Communication (SPI & UART) | Week 9 |
| [Module 10](module10/) | Analog-to-Digital Conversion | Week 10 |
| [Module 11](module11/) | Digital-to-Analog Conversion | Week 11 |

### Advanced Topics (Modules 12-14)
Apply signal processing and hardware acceleration techniques.

| Module | Topic | Duration |
|--------|-------|----------|
| [Module 12](module12/) | Digital Signal Processing & Filtering | Week 12 |
| [Module 13](module13/) | High-Level Synthesis | Week 13 |
| [Module 14](module14/) | System Integration | Week 14-15 |

---

## Prerequisites Map

```
Module 1 ─────► Module 2 ─────► Module 3 ─────► Module 4
                                    │               │
                                    ▼               ▼
                                Module 5 ─────► Module 6 ─────► Module 7
                                                                    │
                                Module 8 ◄──────────────────────────┘
                                    │
                                    ▼
                                Module 9 ─────► Module 10 ─────► Module 11
                                                                     │
                                Module 12 ◄──────────────────────────┘
                                    │
                                    ▼
                                Module 13 ─────► Module 14
```

---

## Recommended Study Path

1. **Complete each module in order** - Concepts build upon previous modules
2. **Do the readings before lecture** - Come prepared with questions
3. **Complete hands-on exercises** - Practice reinforces learning
4. **Start labs early** - Allow time for debugging and office hours
5. **Review before exams** - Use module summaries and practice problems
