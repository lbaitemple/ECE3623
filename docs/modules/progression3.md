---
title: "Progression 3: Timing & Peripherals"
layout: default
nav_order: 3
parent: Course Modules
has_children: true
---

# Progression 3: Timing & Peripherals

Learn software timer management, watchdog implementation, and SPI peripheral interfacing.

---

## Overview

This progression focuses on timing mechanisms and peripheral communication:

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [Module 8](module08/) | Software Timers and Watchdog | xTimerCreate(), one-shot vs auto-reload, watchdog kicking |
| [Module 9](module09/) | ADC and SPI Communication | SPI protocol, PmodAD1, data acquisition |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                TIMING & PERIPHERALS PROGRESSION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│        Module 8                        Module 9                  │
│       ┌──────────┐                   ┌──────────┐               │
│       │ Software │                   │   SPI    │               │
│       │  Timers  │──────────────────►│   ADC    │               │
│       │ Watchdog │                   │Interface │               │
│       └──────────┘                   └──────────┘               │
│                                                                  │
│       Time Management                Peripheral I/O              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Completion of Progression 2 (Synchronization)
- Understanding of queues and interrupt handling

---

## Outcomes

After completing this progression, you will be able to:

1. Create and manage FreeRTOS software timers
2. Implement watchdog timer strategies for system reliability
3. Configure SPI communication on Zynq
4. Interface with external ADC modules (PmodAD1)
