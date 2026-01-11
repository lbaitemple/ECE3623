---
title: "Module 5: Priority Inversion and Inheritance"
layout: default
parent: "Progression 2: Synchronization"
grand_parent: "Course Modules"
nav_order: 2
---

# Module 5: Priority Inversion and Inheritance
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Define priority inversion and explain why it is dangerous
2. Describe the Mars Pathfinder incident and its lessons
3. Explain unbounded priority inversion vs. bounded blocking
4. Implement the Priority Inheritance Protocol
5. Configure FreeRTOS mutexes to use priority inheritance

---

## Overview

Priority inversion is one of the most insidious problems in real-time systems. It occurs when a high-priority task is indirectly blocked by a lower-priority task, violating the fundamental principle that higher-priority work should always execute first. The most famous example—the Mars Pathfinder anomaly—demonstrates how priority inversion can threaten mission-critical systems.

---

## What is Priority Inversion?

### The Problem Defined

**Priority Inversion** occurs when a high-priority task is blocked waiting for a resource held by a low-priority task, while medium-priority tasks preempt the low-priority task.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRIORITY INVERSION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Priority High ────────────────────────────────────────────────  │
│                        │ Task H blocked waiting for mutex       │
│                        │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│          │
│                        │                             │          │
│  Priority Med  ────────┴─────────────────────────────┼────────  │
│                        Task M runs (preempts L)      │          │
│                        ╔═══════════════════════════╗ │          │
│                        ║   M has NO deadline but   ║ │          │
│                        ║   runs BEFORE H !         ║ │          │
│                        ╚═══════════════════════════╝ │          │
│                                                      │          │
│  Priority Low  ─────┬──────────────────────────────┬─┴────────  │
│                     │ L holds mutex                │L resumes   │
│                     └──────────────────────────────┴──────────  │
│                                                                  │
│         Time ────────────────────────────────────────────────►  │
└─────────────────────────────────────────────────────────────────┘
```

### The Sequence of Events

1. **Low-priority task (L)** acquires a mutex
2. **High-priority task (H)** becomes ready and preempts L
3. **H tries to acquire the same mutex** → H blocks (waiting for L)
4. **Medium-priority task (M)** becomes ready
5. **M preempts L** (M has higher priority than L)
6. **H remains blocked** while M runs, even though H has higher priority!

### Unbounded Priority Inversion

The most dangerous aspect: the blocking time is **unbounded**. Any number of medium-priority tasks can preempt L, indefinitely delaying H.

```
┌─────────────────────────────────────────────────────────────────┐
│            UNBOUNDED PRIORITY INVERSION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Task H (High)     │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│        │
│                    │         Blocked indefinitely!      │        │
│                    │                                    │        │
│  Task M3 (Med)     │                              ┌────┘        │
│  Task M2 (Med)     │                    ┌─────────┘             │
│  Task M1 (Med)     │          ┌─────────┘                       │
│                    │          │                                  │
│  Task L (Low)   ───┴──────────┴─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                    │ Holds    Preempted by M1, M2, M3...        │
│                    │ mutex                                       │
│                                                                  │
│  H could be blocked for MINUTES while it should run in μs!      │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Mars Pathfinder Story

### The Mission

In July 1997, NASA's Mars Pathfinder spacecraft successfully landed on Mars. The mission seemed flawless until the lander began experiencing repeated system resets.

![Mars Pathfinder]({{ site.baseurl }}/docs/imgs/pdf_images/freertos/img-001.png)

### The Symptoms

- The spacecraft would periodically reset
- Resets occurred during data-gathering operations
- The problem was intermittent and hard to reproduce
- Mission controllers were deeply concerned

### The Root Cause: Priority Inversion

The Pathfinder software ran on VxWorks RTOS with three key tasks:

| Task | Priority | Function |
|------|----------|----------|
| **Bus Management** | High | Manage 1553 data bus, deadline-critical |
| **Communication** | Medium | Handle ground communication |
| **Meteorological** | Low | Collect weather data |

The meteorological task and bus management task shared access to a data structure protected by a mutex.

### The Failure Scenario

```
┌─────────────────────────────────────────────────────────────────┐
│                MARS PATHFINDER FAILURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Meteorological task (LOW) acquires mutex                    │
│                                                                  │
│  2. Bus Management task (HIGH) wakes up, needs mutex            │
│     → Blocks waiting for LOW to release mutex                   │
│                                                                  │
│  3. Communication task (MEDIUM) becomes ready                   │
│     → Preempts LOW task                                         │
│     → HIGH task still blocked!                                  │
│                                                                  │
│  4. Bus Management deadline MISSED                              │
│     → Watchdog timer detects deadline miss                      │
│     → SYSTEM RESET!                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Fix

JPL engineers uploaded a software patch that enabled **priority inheritance** on the mutex. This fix was applied via a remote software update—from 119 million miles away!

### Lessons Learned

1. **Priority inversion can occur in any RTOS** if not properly managed
2. **Testing may not reveal the problem** (it was intermittent)
3. **Priority inheritance should be enabled by default** for mutexes protecting shared resources
4. **Watchdog timers saved the mission** by recovering from the failure

---

## Priority Inheritance Protocol

### The Solution

**Priority Inheritance** temporarily raises the priority of a low-priority task when it blocks a higher-priority task.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│            WITH PRIORITY INHERITANCE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Priority High ────────────────────────────────────────────────  │
│                        │H blocked  │H runs (got mutex)          │
│                        │           │                            │
│                     ┌──┴───────────┼────────────────┐           │
│                     │ L inherits   │                │           │
│                     │ H's priority!│                │           │
│                     └──────────────┤                │           │
│                                    │                │           │
│  Priority Med  ────────────────────┼─ ─ ─ ─ ─ ─ ─ ─│────────── │
│                     M cannot       │  M waits      │  M runs   │
│                     preempt L!     │               │           │
│                                    │               │           │
│  Priority Low  ─────┬──────────────┴───────────────┴─────────   │
│                     │ L runs at High priority       │L normal  │
│                     └───────────────────────────────┘           │
│                                                                  │
│         Time ────────────────────────────────────────────────►  │
└─────────────────────────────────────────────────────────────────┘
```

### The Protocol Steps

1. **Task L** (low priority) acquires mutex M
2. **Task H** (high priority) attempts to acquire mutex M
3. **H blocks** because L holds M
4. **Kernel elevates L's priority** to match H's priority
5. **L runs at high priority** until it releases M
6. **L releases M** → L's priority restored to original
7. **H acquires M** and runs immediately

### Key Benefits

| Aspect | Without Inheritance | With Inheritance |
|--------|---------------------|------------------|
| Blocking time | Unbounded | Bounded (critical section length) |
| Medium task interference | Yes | No |
| High task deadline | May miss | Protected |

---

## Priority Inheritance in FreeRTOS

### FreeRTOS Mutexes with Priority Inheritance

**Good news**: FreeRTOS mutexes implement priority inheritance by default!

```c
#include "FreeRTOS.h"
#include "semphr.h"

SemaphoreHandle_t xMutex;

// Create a mutex - priority inheritance is AUTOMATIC
void init(void) {
    xMutex = xSemaphoreCreateMutex();
    // Priority inheritance is built-in!
}
```

### Configuration Requirements

Priority inheritance requires these settings in `FreeRTOSConfig.h`:

```c
// Required for priority inheritance
#define configUSE_MUTEXES                1

// Optional: enable priority inheritance statistics
#define INCLUDE_uxTaskPriorityGet        1
#define INCLUDE_vTaskPrioritySet         1
```

### Demonstration Example

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include "xil_printf.h"

SemaphoreHandle_t xSharedMutex;

// Low priority task
void vLowPriorityTask(void *param) {
    for (;;) {
        xil_printf("LOW: Attempting to take mutex...\r\n");
        
        if (xSemaphoreTake(xSharedMutex, portMAX_DELAY) == pdTRUE) {
            xil_printf("LOW: Got mutex, priority = %d\r\n", 
                       uxTaskPriorityGet(NULL));
            
            // Simulate long critical section
            for (int i = 0; i < 10; i++) {
                vTaskDelay(pdMS_TO_TICKS(100));
                xil_printf("LOW: Working... priority = %d\r\n",
                           uxTaskPriorityGet(NULL));
            }
            
            xSemaphoreGive(xSharedMutex);
            xil_printf("LOW: Released mutex, priority = %d\r\n",
                       uxTaskPriorityGet(NULL));
        }
        
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

// High priority task
void vHighPriorityTask(void *param) {
    vTaskDelay(pdMS_TO_TICKS(200));  // Let low task get mutex first
    
    for (;;) {
        xil_printf("HIGH: Attempting to take mutex...\r\n");
        
        // This will trigger priority inheritance!
        if (xSemaphoreTake(xSharedMutex, portMAX_DELAY) == pdTRUE) {
            xil_printf("HIGH: Got mutex!\r\n");
            
            // Quick critical section
            vTaskDelay(pdMS_TO_TICKS(50));
            
            xSemaphoreGive(xSharedMutex);
            xil_printf("HIGH: Released mutex\r\n");
        }
        
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

// Medium priority task (would cause priority inversion without inheritance)
void vMediumPriorityTask(void *param) {
    vTaskDelay(pdMS_TO_TICKS(300));  // Let scenario develop
    
    for (;;) {
        xil_printf("MEDIUM: Running (no mutex needed)...\r\n");
        
        // Busy work - would block LOW without priority inheritance
        for (volatile int i = 0; i < 1000000; i++);
        
        xil_printf("MEDIUM: Done\r\n");
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main(void) {
    xSharedMutex = xSemaphoreCreateMutex();
    
    xTaskCreate(vLowPriorityTask, "Low", 512, NULL, 1, NULL);
    xTaskCreate(vMediumPriorityTask, "Med", 512, NULL, 2, NULL);
    xTaskCreate(vHighPriorityTask, "High", 512, NULL, 3, NULL);
    
    vTaskStartScheduler();
    for (;;);
}
```

### Expected Output

```
LOW: Attempting to take mutex...
LOW: Got mutex, priority = 1
LOW: Working... priority = 1
HIGH: Attempting to take mutex...           <- HIGH blocks, triggers inheritance
LOW: Working... priority = 3                <- LOW now runs at priority 3!
LOW: Working... priority = 3
MEDIUM: Running (no mutex needed)...        <- MEDIUM can't preempt (LOW at 3)
LOW: Working... priority = 3
...
LOW: Released mutex, priority = 1           <- Priority restored
HIGH: Got mutex!
HIGH: Released mutex
MEDIUM: Done
```

---

## Priority Ceiling Protocol

An alternative to priority inheritance:

### How Priority Ceiling Works

1. Each mutex is assigned a "ceiling priority" = highest priority of any task that uses it
2. When a task acquires a mutex, its priority is immediately raised to the ceiling
3. No other task can preempt while holding the mutex (if properly configured)

### Comparison

| Aspect | Priority Inheritance | Priority Ceiling |
|--------|---------------------|------------------|
| When priority raised | Only when blocking occurs | Immediately on acquire |
| Implementation | Reactive | Proactive |
| Deadlock prevention | No | Yes (if all mutexes use ceiling) |
| Complexity | Lower | Higher |
| FreeRTOS support | Yes (default) | No (manual implementation) |

---

## Best Practices

### Avoiding Priority Inversion

1. **Use FreeRTOS mutexes** (not binary semaphores) for resource protection
2. **Keep critical sections short** to minimize blocking time
3. **Avoid nested mutex acquisition** when possible
4. **Use consistent lock ordering** if multiple mutexes needed
5. **Consider task design** - can the resource be accessed by only one task?

### When Priority Inheritance Isn't Enough

| Scenario | Problem | Solution |
|----------|---------|----------|
| Very long critical sections | Blocking still too long | Redesign data access |
| Multiple blocking tasks | Chain of inheritance | Priority ceiling or redesign |
| ISR needs resource | ISRs can't use mutexes | Use deferred interrupt handling |

---

## Lecture Materials

- [Priority Inversion and Inheritance]({{ site.baseurl }}/docs/pdfs/FreeRTOS.pdf)
- [Week 5 Slides]({{ site.baseurl }}/docs/pdfs/Week%205.pdf)
- [Mars Pathfinder Case Study](https://www.cs.unc.edu/~anderson/teach/comp790/papers/mars_pathfinder_long_version.html)

---

## Reading Assignments

1. *Mastering the FreeRTOS Real Time Kernel*, Chapter 7: Mutexes and Priority Inheritance
2. "What Really Happened on Mars?" - Glenn Reeves, JPL
3. Sha, L., Rajkumar, R., & Lehoczky, J. P. (1990). Priority inheritance protocols

---

## Practice Questions

1. Define priority inversion in your own words. Why is it dangerous?
2. In the Mars Pathfinder incident, which three tasks were involved and what were their priorities?
3. Explain how priority inheritance prevents unbounded priority inversion.
4. A system has tasks at priorities 1, 3, and 5. Task 1 holds a mutex. Task 5 blocks on the mutex. What priority does Task 1 inherit?
5. Why doesn't FreeRTOS implement priority ceiling protocol?
6. What happens to the inherited priority when the mutex is released?
7. Why should you use a mutex instead of a binary semaphore when protecting shared resources?

---

## Summary

Priority inversion is a subtle but dangerous problem where high-priority tasks are blocked by lower-priority tasks through mutex interactions, while medium-priority tasks continue to run. The Mars Pathfinder incident demonstrated how priority inversion can cause mission-critical failures. FreeRTOS mutexes implement priority inheritance by default, which bounds the blocking time by temporarily elevating the priority of the mutex holder. Understanding and properly using these mechanisms is essential for reliable real-time system design.

---

## Next Module

[Module 6: Hardware Interrupts and GIC →]({{ site.baseurl }}/docs/modules/module06.html)

2. Calculate the prescaler and count value needed for a 20ms interrupt period with a 100 MHz clock and 16-bit counter.

3. What is the difference between free-running and periodic timer modes?

4. A 16-bit counter at 1 MHz overflows. How long does it take to overflow?

5. Why might you use a PL-based timer instead of a PS timer?

---

## Summary

Counter-timers are fundamental hardware peripherals for embedded systems. We explored timer architecture, prescaler calculations, operating modes, and implementation on both Zynq PS and PL. These concepts enable precise timing control essential for real-time applications. The next module covers timer-based measurements and watchdog timers for system reliability.

---

## Next Module

[Module 6: Timer Measurement & Watchdog →]({{ site.baseurl }}/docs/modules/module06.html)
