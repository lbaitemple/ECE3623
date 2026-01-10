---
title: "Progression 1: Foundations"
layout: default
nav_order: 1
parent: Course Modules
has_children: true
---

# Progression 1: Foundations

Build your understanding of embedded systems architecture, real-time computing concepts, and basic RTOS task management.

---

## Overview

This progression introduces the core concepts needed for embedded systems development on the Zynq platform:

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [Module 1](module01/) | Embedded SoC and Zynq Architecture | PS/PL architecture, ARM Cortex-A9, hard vs soft cores |
| [Module 2](module02/) | RTOS Fundamentals | Real-time systems, hard vs soft real-time, FreeRTOS introduction |
| [Module 3](module03/) | FreeRTOS Task Management | Task states, xTaskCreate(), priorities, scheduling |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                    FOUNDATIONS PROGRESSION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Module 1                Module 2                Module 3       │
│  ┌──────────┐           ┌──────────┐           ┌──────────┐     │
│  │   Zynq   │           │   RTOS   │           │   Task   │     │
│  │  SoC     │──────────►│Fundamentals──────────►│Management│     │
│  │Architecture          │          │           │          │     │
│  └──────────┘           └──────────┘           └──────────┘     │
│                                                                  │
│  Hardware                Concepts               Implementation   │
│  Platform                                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Basic C programming
- Familiarity with digital logic concepts
- Understanding of computer architecture basics

---

## Outcomes

After completing this progression, you will be able to:

1. Describe the Zynq-7000 AP SoC architecture and PS-PL integration
2. Explain real-time system requirements and RTOS benefits
3. Create and manage FreeRTOS tasks with appropriate priorities
4. Understand preemptive scheduling and task states
