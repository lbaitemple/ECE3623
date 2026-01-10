---
title: "Module 3: FreeRTOS Task Management"
layout: default
parent: "Progression 1: Foundations"
grand_parent: "Course Modules"
nav_order: 3
---

# Module 3: FreeRTOS Task Management
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Describe the four FreeRTOS task states: Running, Ready, Blocked, and Suspended
2. Create tasks using `xTaskCreate()` with appropriate parameters
3. Assign and manage task priorities effectively
4. Distinguish between pre-emptive and time-slicing scheduling
5. Implement task control operations (delay, suspend, resume, delete)

---

## Overview

Tasks are the fundamental building blocks of any FreeRTOS application. Each task is an independent thread of execution with its own stack and context. Understanding task states, priorities, and scheduling is essential for building reliable real-time systems.

---

## Task States

Every FreeRTOS task exists in one of four states at any given time:

### The Four Task States

```
                    ┌─────────────┐
         Event      │  SUSPENDED  │
        ┌──────────►│             │◄──────────┐
        │           └─────────────┘           │
        │                 │                   │
        │          vTaskResume()              │ vTaskSuspend()
        │                 │                   │
        │                 ▼                   │
        │           ┌─────────────┐           │
        │   ┌──────►│    READY    │◄──────────┼─────────┐
        │   │       └─────────────┘           │         │
        │   │             │                   │         │
        │   │     Scheduler selects           │     Preempted
        │   │             │                   │         │
        │   │             ▼                   │         │
        │   │       ┌─────────────┐           │         │
        │   │       │   RUNNING   │───────────┴─────────┘
        │   │       └─────────────┘
        │   │             │
        │   │        Blocking API call
        │   │        (delay, queue, semaphore)
        │   │             │
        │   │             ▼
        │   │       ┌─────────────┐
        │   └───────│   BLOCKED   │
        │ Event     └─────────────┘
        │ occurs          │
        └─────────────────┘
```

### State Descriptions

| State | Description | Transition Trigger |
|-------|-------------|-------------------|
| **Running** | Currently executing on the CPU | Selected by scheduler |
| **Ready** | Able to run, waiting for CPU | Higher priority task blocks/completes |
| **Blocked** | Waiting for an event or timeout | Blocking API call |
| **Suspended** | Not available for scheduling | `vTaskSuspend()` called |

### Key Points

- **Only ONE task** can be in the Running state at any time (per CPU)
- Tasks in **Ready** state are sorted by priority
- **Blocked** tasks consume no CPU cycles
- **Suspended** tasks must be explicitly resumed

---

## Creating Tasks with xTaskCreate()

### Function Signature

```c
BaseType_t xTaskCreate(
    TaskFunction_t pvTaskCode,     // Pointer to task function
    const char * const pcName,     // Task name (for debugging)
    uint16_t usStackDepth,         // Stack size (in words, not bytes!)
    void *pvParameters,            // Parameter passed to task
    UBaseType_t uxPriority,        // Task priority
    TaskHandle_t *pxCreatedTask    // Handle to created task (optional)
);
```

### Return Value

| Value | Meaning |
|-------|---------|
| `pdPASS` | Task created successfully |
| `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY` | Insufficient heap memory |

### Complete Task Creation Example

```c
#include "FreeRTOS.h"
#include "task.h"
#include "xil_printf.h"

// Task handle - allows control of task from elsewhere
TaskHandle_t xSensorTaskHandle = NULL;

// Task function - must never return
void vSensorTask(void *pvParameters)
{
    // Cast parameter if needed
    int sensorId = (int)pvParameters;
    
    // Infinite loop - tasks must never exit
    for (;;)
    {
        xil_printf("Sensor %d reading...\r\n", sensorId);
        
        // Periodic execution: release CPU for 100ms
        vTaskDelay(pdMS_TO_TICKS(100));
    }
    
    // Should never reach here
    // If task must end, use vTaskDelete(NULL)
}

int main(void)
{
    BaseType_t xReturned;
    
    // Create the sensor task
    xReturned = xTaskCreate(
        vSensorTask,                    // Task function
        "SensorTask",                   // Task name (max 16 chars)
        configMINIMAL_STACK_SIZE * 2,   // Stack size: 400 words
        (void *)1,                      // Parameter: sensor ID = 1
        tskIDLE_PRIORITY + 2,           // Priority: 2
        &xSensorTaskHandle              // Store handle
    );
    
    // Check if task was created successfully
    if (xReturned != pdPASS)
    {
        xil_printf("Failed to create task!\r\n");
        return -1;
    }
    
    // Start the scheduler
    vTaskStartScheduler();
    
    // Should never reach here
    for (;;);
}
```

### Stack Size Guidelines

| Task Type | Recommended Stack | Notes |
|-----------|-------------------|-------|
| Simple task (no function calls) | 128-200 words | Minimal |
| Task with printf/xil_printf | 256-512 words | Printf uses stack heavily |
| Task with deep call chains | 512-1024 words | Analyze with stack profiling |
| Task with ISR-safe APIs | Add 64+ words | ISR context saved on task stack |

**Note**: Stack size is in **words** (4 bytes on ARM), not bytes!

---

## Task Priorities

### Priority Numbering

```c
// Priority 0 is the lowest (idle task priority)
#define tskIDLE_PRIORITY    0

// configMAX_PRIORITIES - 1 is the highest priority
// Typical configuration: priorities 0-6 (configMAX_PRIORITIES = 7)
```

### Priority Assignment Strategy

| Priority Level | Task Type | Examples |
|----------------|-----------|----------|
| Highest (6) | Critical interrupts/emergencies | Safety shutdowns, alarms |
| High (5) | Time-critical control | Motor control, servo updates |
| Medium-High (4) | Sensor acquisition | ADC sampling, encoder reading |
| Medium (3) | Processing tasks | Filtering, calculations |
| Medium-Low (2) | Communication | UART, SPI, network |
| Low (1) | User interface | Display updates, logging |
| Lowest (0) | Idle/housekeeping | Background cleanup, sleep |

### Setting and Changing Priority

```c
// Set priority at creation
xTaskCreate(vTask, "Task", 256, NULL, 3, &xHandle);

// Change priority dynamically
vTaskPrioritySet(xHandle, 5);  // Set to priority 5
vTaskPrioritySet(NULL, 4);     // NULL = current task

// Get current priority
UBaseType_t uxPriority = uxTaskPriorityGet(xHandle);
```

---

## Scheduling: Pre-emptive vs Time-Slicing

### Pre-emptive Scheduling

When `configUSE_PREEMPTION = 1`, a higher-priority task that becomes ready will **immediately preempt** the currently running task.

```
Priority
    ↑
  3 │    ┌────┐              ┌────┐
    │    │ T1 │              │ T1 │
  2 │────┴────┴──────────────┴────┴────────
    │         ┌──────────────┐
    │         │      T2      │
  1 │─────────┴──────────────┴─────────────
    └──────────────────────────────────────► Time
    
    T1 (priority 3) preempts T2 (priority 2) when T1 becomes ready
```

### Time-Slicing (Round-Robin)

When `configUSE_TIME_SLICING = 1`, tasks with **equal priority** share CPU time in turns.

```
    ┌──────────────────────────────────────────────────────────┐
    │                  Same Priority Tasks                      │
    ├────────┬────────┬────────┬────────┬────────┬────────────┤
    │   T1   │   T2   │   T3   │   T1   │   T2   │   T3       │
    ├────────┴────────┴────────┴────────┴────────┴────────────┤
    │                      Time →                               │
    │         Each tick, scheduler rotates among equal tasks    │
    └──────────────────────────────────────────────────────────┘
```

### Configuration in FreeRTOSConfig.h

```c
/* Pre-emptive scheduling: higher priority preempts lower */
#define configUSE_PREEMPTION         1

/* Time-slicing: round-robin among equal priorities */
#define configUSE_TIME_SLICING       1

/* Both enabled = most common configuration */
```

### Scheduling Decision Points

The scheduler runs at:
1. Each tick interrupt
2. When a task calls a blocking API
3. When a task calls `taskYIELD()`
4. When an ISR wakes a higher-priority task

---

## Task Control Operations

### Task Delay

```c
// Relative delay: delay from NOW
vTaskDelay(pdMS_TO_TICKS(100));  // Delay 100ms from now

// Absolute delay: maintain precise period
TickType_t xLastWakeTime = xTaskGetTickCount();
const TickType_t xPeriod = pdMS_TO_TICKS(100);

for (;;)
{
    doWork();
    
    // Delay until exactly 100ms after last wake
    vTaskDelayUntil(&xLastWakeTime, xPeriod);
}
```

**Difference Illustrated**:

```
vTaskDelay(100):
    ├── Execute (40ms) ──┼── Delay (100ms) ──┼── Execute (40ms) ──┤
    Total period: 140ms (execution + delay)

vTaskDelayUntil(&last, 100):
    ├── Execute (40ms) ──┼── Delay (60ms) ──┼── Execute (40ms) ──┤
    Total period: 100ms (fixed period, compensates for execution time)
```

### Suspend and Resume

```c
// Suspend a task - removes from ready list
vTaskSuspend(xTaskHandle);    // Suspend specific task
vTaskSuspend(NULL);           // Suspend current task

// Resume a task - returns to ready state
vTaskResume(xTaskHandle);

// Resume from ISR (interrupt-safe version)
BaseType_t xHigherPriorityWoken = pdFALSE;
xTaskResumeFromISR(xTaskHandle);
portYIELD_FROM_ISR(xHigherPriorityWoken);
```

### Delete Task

```c
// Delete a task (use with caution!)
vTaskDelete(xTaskHandle);  // Delete specific task
vTaskDelete(NULL);         // Delete current task (self-delete)

// After deletion:
// - Stack memory is freed by idle task
// - Task handle becomes invalid
// - Never delete a task holding a mutex!
```

---

## Task Implementation Best Practices

### The Correct Task Pattern

```c
void vCorrectTask(void *pvParameters)
{
    // 1. One-time initialization
    initializePeripherals();
    
    // 2. Infinite loop (tasks must never return!)
    for (;;)
    {
        // 3. Do work
        processData();
        
        // 4. Block to release CPU (CRITICAL!)
        vTaskDelay(pdMS_TO_TICKS(10));
    }
    
    // 5. Cleanup if task might be deleted (rare)
    // vTaskDelete(NULL);
}
```

### Common Mistakes to Avoid

```c
// WRONG: Task that returns
void vBadTask1(void *param)
{
    doSomething();
    return;  // CRASH! Tasks must never return
}

// WRONG: Busy-wait loop
void vBadTask2(void *param)
{
    for (;;)
    {
        while (!dataReady);  // Burns CPU cycles!
        processData();
    }
}

// WRONG: Blocking without delay
void vBadTask3(void *param)
{
    for (;;)
    {
        pollSensor();  // If sensor always ready, starves lower priority tasks
    }
}
```

---

## Practical Example: Multi-Task System

```c
#include "FreeRTOS.h"
#include "task.h"
#include "xil_printf.h"
#include "xgpio.h"

// Task handles
TaskHandle_t xBlinkHandle;
TaskHandle_t xMonitorHandle;
TaskHandle_t xControlHandle;

// Blink LED task - low priority, runs continuously
void vBlinkTask(void *param)
{
    XGpio *pGpio = (XGpio *)param;
    uint8_t ledState = 0;
    
    for (;;)
    {
        ledState ^= 1;
        XGpio_DiscreteWrite(pGpio, 1, ledState);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

// Monitor task - medium priority, periodic sampling
void vMonitorTask(void *param)
{
    TickType_t xLastWake = xTaskGetTickCount();
    
    for (;;)
    {
        int temp = readTemperature();
        xil_printf("Temperature: %d\r\n", temp);
        
        // Precise 100ms period
        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(100));
    }
}

// Control task - high priority, fast response
void vControlTask(void *param)
{
    for (;;)
    {
        if (checkAlarm())
        {
            activateSafetyShutdown();
        }
        vTaskDelay(pdMS_TO_TICKS(10));  // 10ms loop
    }
}

int main(void)
{
    static XGpio gpio;
    XGpio_Initialize(&gpio, XPAR_AXI_GPIO_0_DEVICE_ID);
    
    // Create tasks with appropriate priorities
    xTaskCreate(vBlinkTask, "Blink", 256, &gpio, 1, &xBlinkHandle);
    xTaskCreate(vMonitorTask, "Monitor", 512, NULL, 2, &xMonitorHandle);
    xTaskCreate(vControlTask, "Control", 256, NULL, 4, &xControlHandle);
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Lecture Materials

- [FreeRTOS Task Management]({{ site.baseurl }}/ece3623/FreeRTOS.pdf)
- [Week 3 Slides]({{ site.baseurl }}/ece3623/Week%203.pdf)
- [Week 3 Recap]({{ site.baseurl }}/ece3623/Week%203-R.pdf)

---

## Reading Assignments

1. *Mastering the FreeRTOS Real Time Kernel*, Chapter 3: Task Management
2. FreeRTOS Documentation: Task API Reference
3. FreeRTOS Documentation: Scheduling Algorithm

---

## Practice Questions

1. Draw the FreeRTOS task state diagram and label all transitions.
2. What happens if you specify a stack size of 256 for `xTaskCreate()`? How many bytes is that?
3. Explain the difference between `vTaskDelay()` and `vTaskDelayUntil()`. When would you use each?
4. A system has three tasks at priorities 1, 2, and 3. Task 3 is blocked. Task 2 is running. What happens when Task 3 unblocks?
5. What is the consequence of having `configUSE_PREEMPTION = 0`?
6. Why must FreeRTOS tasks contain an infinite loop? What happens if a task function returns?
7. A task with priority 5 creates a new task with priority 3. Which task runs immediately after `xTaskCreate()` returns?

---

## Summary

This module covered FreeRTOS task management fundamentals. Tasks exist in four states (Running, Ready, Blocked, Suspended) and transition between states based on scheduling decisions and API calls. The `xTaskCreate()` function creates tasks with specific priorities and stack allocations. FreeRTOS supports both pre-emptive and time-slicing scheduling modes. Proper task design—with infinite loops, appropriate blocking, and correct priority assignment—is essential for building responsive real-time systems.

---

## Next Module

[Module 4: Mutual Exclusion and Resource Sharing →]({{ site.baseurl }}/docs/modules/module04/)
