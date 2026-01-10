---
title: "Progression 2: Synchronization"
layout: default
nav_order: 2
parent: Course Modules
has_children: true
---

# Progression 2: Synchronization & Communication

Master the techniques for safe resource sharing, inter-task communication, and hardware interrupt handling.

---

## Overview

This progression covers critical concepts for building reliable multi-tasking systems:

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [Module 4](module04/) | Mutual Exclusion | Critical sections, mutexes, race conditions |
| [Module 5](module05/) | Priority Inversion | Mars Pathfinder, priority inheritance protocol |
| [Module 6](module06/) | Hardware Interrupts | GIC, ISR design, deferred interrupt handling |
| [Module 7](module07/) | Queues | Inter-task communication, producer-consumer pattern |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│               SYNCHRONIZATION PROGRESSION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Module 4              Module 5              Module 6           │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐       │
│  │  Mutual  │          │ Priority │          │ Hardware │       │
│  │Exclusion │─────────►│Inversion │─────────►│Interrupts│       │
│  │          │          │          │          │          │       │
│  └──────────┘          └──────────┘          └──────────┘       │
│       │                                            │             │
│       │                                            │             │
│       └────────────────┬───────────────────────────┘             │
│                        ▼                                         │
│                   ┌──────────┐                                   │
│                   │  Queues  │  Module 7                         │
│                   │          │                                   │
│                   └──────────┘                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Completion of Progression 1 (Foundations)
- Understanding of FreeRTOS task creation and states

---

## Outcomes

After completing this progression, you will be able to:

1. Protect shared resources using critical sections and mutexes
2. Identify and prevent priority inversion scenarios
3. Configure and handle hardware interrupts on Zynq
4. Implement thread-safe inter-task communication using queues
