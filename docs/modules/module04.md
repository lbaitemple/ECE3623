---
title: "Module 4: Mutual Exclusion and Resource Sharing"
layout: default
parent: "Progression 2: Synchronization"
grand_parent: "Course Modules"
nav_order: 1
---

# Module 4: Mutual Exclusion and Resource Sharing
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Identify the problems caused by concurrent access to shared resources
2. Implement critical sections using `taskENTER_CRITICAL()` and `taskEXIT_CRITICAL()`
3. Use FreeRTOS mutexes for resource protection
4. Choose the appropriate synchronization mechanism for different scenarios
5. Avoid common pitfalls such as deadlock and starvation

---

## Overview

In multi-tasking systems, tasks often need to share resources: global variables, hardware peripherals, memory buffers, or communication channels. Without proper protection, concurrent access leads to **race conditions** and **data corruption**. This module covers the techniques FreeRTOS provides to ensure safe resource sharing.

---

## The Shared Resource Problem

### What Can Go Wrong?

Consider two tasks incrementing a shared counter:

```c
volatile int shared_counter = 0;

void TaskA(void *param) {
    for (;;) {
        shared_counter++;  // Read-Modify-Write operation
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void TaskB(void *param) {
    for (;;) {
        shared_counter++;  // Same operation
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

### The Race Condition

The `shared_counter++` operation is NOT atomic. It consists of three steps:

```
┌─────────────────────────────────────────────────────────────────┐
│                    shared_counter++ expands to:                  │
├─────────────────────────────────────────────────────────────────┤
│  1. LOAD:  Read current value from memory into register         │
│  2. ADD:   Increment the register value                         │
│  3. STORE: Write new value back to memory                       │
└─────────────────────────────────────────────────────────────────┘
```

### Race Condition Scenario

```
                 shared_counter = 5
                        │
    Task A              │              Task B
    ──────              │              ──────
1. LOAD r1, [counter]   │                      (r1 = 5)
                        │
        ─── PREEMPTION ─┼──────────────────────────►
                        │
                        │  1. LOAD r1, [counter]  (r1 = 5)
                        │  2. ADD r1, #1          (r1 = 6)
                        │  3. STORE [counter], r1 (counter = 6)
        ◄───────────────┼── PREEMPTION ───────────
                        │
2. ADD r1, #1           │                      (r1 = 6)
3. STORE [counter], r1  │                      (counter = 6)
                        │
                 shared_counter = 6  ← Should be 7!
```

**Result**: One increment was lost. The counter shows 6 instead of 7.

---

## Critical Sections

A **critical section** is a region of code that must execute atomically—without interruption from other tasks or interrupts.

### taskENTER_CRITICAL() and taskEXIT_CRITICAL()

These macros disable interrupts to create an uninterruptible section:

```c
volatile int shared_counter = 0;

void TaskA(void *param) {
    for (;;) {
        taskENTER_CRITICAL();
        // --- Critical Section Start ---
        shared_counter++;
        // --- Critical Section End ---
        taskEXIT_CRITICAL();
        
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  taskENTER_CRITICAL()                                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. Disable interrupts (raises interrupt mask)            │  │
│  │  2. Increment nesting counter                             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ... Your critical section code (interrupts disabled) ...       │
│                                                                  │
│  taskEXIT_CRITICAL()                                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. Decrement nesting counter                             │  │
│  │  2. If counter == 0, re-enable interrupts                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Nesting Support

Critical sections can be nested safely:

```c
void OuterFunction(void) {
    taskENTER_CRITICAL();  // Nesting level = 1
    
    InnerFunction();       // Can also use critical sections
    
    taskEXIT_CRITICAL();   // Nesting level = 0, interrupts restored
}

void InnerFunction(void) {
    taskENTER_CRITICAL();  // Nesting level = 2
    
    // Do something...
    
    taskEXIT_CRITICAL();   // Nesting level = 1 (interrupts still disabled)
}
```

### Critical Section Guidelines

| Do | Don't |
|----|-------|
| Keep critical sections as short as possible | Block or delay inside critical section |
| Protect only shared data access | Call FreeRTOS API functions |
| Use for simple read-modify-write | Perform lengthy computations |
| Match every ENTER with EXIT | Use for protecting long operations |

### ISR-Safe Version

For use within interrupt service routines:

```c
void MyISRHandler(void) {
    UBaseType_t uxSavedInterruptStatus;
    
    uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
    
    // Critical section in ISR
    shared_data++;
    
    taskEXIT_CRITICAL_FROM_ISR(uxSavedInterruptStatus);
}
```

---

## Mutexes (Mutual Exclusion Semaphores)

When critical sections are too restrictive (they disable ALL interrupts), mutexes provide a better solution for longer operations.

### What is a Mutex?

A **mutex** is a special type of binary semaphore with ownership:
- Only ONE task can "hold" the mutex at a time
- The task that takes the mutex must be the one to give it back
- Supports priority inheritance (covered in Module 5)

### Creating and Using a Mutex

```c
#include "FreeRTOS.h"
#include "semphr.h"

// Global mutex handle
SemaphoreHandle_t xUartMutex;

// Create mutex during initialization
void initMutex(void) {
    xUartMutex = xSemaphoreCreateMutex();
    
    if (xUartMutex == NULL) {
        // Failed to create mutex - insufficient heap
        xil_printf("Error: Could not create mutex\r\n");
    }
}

// Task using the mutex-protected resource
void TaskA(void *param) {
    for (;;) {
        // Attempt to take mutex, wait up to 100ms
        if (xSemaphoreTake(xUartMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            
            // --- Mutex held: safe to access UART ---
            xil_printf("Task A: Sending data...\r\n");
            sendUartData(buffer, len);
            xil_printf("Task A: Done\r\n");
            // --- End of protected section ---
            
            // MUST give mutex back!
            xSemaphoreGive(xUartMutex);
        }
        else {
            // Timeout - could not acquire mutex
            xil_printf("Task A: Could not get UART access\r\n");
        }
        
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

### Mutex API Summary

| Function | Description |
|----------|-------------|
| `xSemaphoreCreateMutex()` | Create a mutex |
| `xSemaphoreTake(mutex, timeout)` | Acquire mutex (block if unavailable) |
| `xSemaphoreGive(mutex)` | Release mutex |
| `xSemaphoreGetMutexHolder(mutex)` | Get handle of task holding mutex |

### Mutex vs Critical Section

| Aspect | Critical Section | Mutex |
|--------|------------------|-------|
| **Mechanism** | Disable interrupts | Task blocking |
| **Duration** | Very short (microseconds) | Can be longer (milliseconds) |
| **Other tasks** | Cannot run | Can run (if not waiting for mutex) |
| **Interrupts** | Disabled | Enabled |
| **Use case** | Simple variable protection | Peripheral access, buffers |
| **Priority inheritance** | N/A | Supported |

---

## Protecting Peripheral Access

### Example: Shared SPI Bus

Multiple tasks need to access devices on the same SPI bus:

```c
SemaphoreHandle_t xSpiMutex;

// Initialize SPI mutex
void SPI_Init(void) {
    xSpiMutex = xSemaphoreCreateMutex();
    configASSERT(xSpiMutex != NULL);
    
    // Initialize SPI hardware...
}

// Temperature sensor task
void TempSensorTask(void *param) {
    float temperature;
    
    for (;;) {
        if (xSemaphoreTake(xSpiMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            // Select temperature sensor chip
            GPIO_SetChipSelect(TEMP_SENSOR_CS);
            
            // Read temperature via SPI
            SPI_Transfer(cmd, response, 2);
            temperature = ConvertToTemp(response);
            
            // Deselect chip
            GPIO_ClearChipSelect(TEMP_SENSOR_CS);
            
            xSemaphoreGive(xSpiMutex);
        }
        
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// ADC reading task (same SPI bus, different device)
void ADCTask(void *param) {
    uint16_t adc_value;
    
    for (;;) {
        if (xSemaphoreTake(xSpiMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            GPIO_SetChipSelect(ADC_CS);
            
            SPI_Transfer(adc_cmd, adc_response, 3);
            adc_value = ParseADCValue(adc_response);
            
            GPIO_ClearChipSelect(ADC_CS);
            
            xSemaphoreGive(xSpiMutex);
        }
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## Deadlock

### What is Deadlock?

**Deadlock** occurs when two or more tasks are permanently blocked, each waiting for a resource held by the other.

```
┌──────────────────────────────────────────────────────────────┐
│                        DEADLOCK                               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│     Task A                              Task B                │
│        │                                   │                  │
│        ▼                                   ▼                  │
│   Take Mutex1 ───────────────────── Take Mutex2              │
│        │                                   │                  │
│        ▼                                   ▼                  │
│   Take Mutex2 ◄─────── BLOCKED ──────► Take Mutex1           │
│    (wait...)                            (wait...)             │
│        │                                   │                  │
│        └───────── Neither can proceed ─────┘                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Deadlock Example

```c
// DANGEROUS: This can cause deadlock!
SemaphoreHandle_t xMutexA, xMutexB;

void TaskX(void *param) {
    for (;;) {
        xSemaphoreTake(xMutexA, portMAX_DELAY);
        vTaskDelay(1);  // Context switch could happen here
        xSemaphoreTake(xMutexB, portMAX_DELAY);
        
        // Use both resources...
        
        xSemaphoreGive(xMutexB);
        xSemaphoreGive(xMutexA);
    }
}

void TaskY(void *param) {
    for (;;) {
        xSemaphoreTake(xMutexB, portMAX_DELAY);  // Opposite order!
        vTaskDelay(1);
        xSemaphoreTake(xMutexA, portMAX_DELAY);
        
        // Use both resources...
        
        xSemaphoreGive(xMutexA);
        xSemaphoreGive(xMutexB);
    }
}
```

### Deadlock Prevention Strategies

| Strategy | Description |
|----------|-------------|
| **Lock ordering** | Always acquire mutexes in the same order |
| **Timeout** | Use timeout instead of `portMAX_DELAY` |
| **Try-lock** | Non-blocking attempt, release and retry if fails |
| **Single lock** | Use one mutex to protect multiple resources |

### Safe Version

```c
// SAFE: Consistent lock ordering
void TaskX(void *param) {
    for (;;) {
        xSemaphoreTake(xMutexA, portMAX_DELAY);  // Always A first
        xSemaphoreTake(xMutexB, portMAX_DELAY);  // Then B
        
        // Use both resources...
        
        xSemaphoreGive(xMutexB);  // Release in reverse order
        xSemaphoreGive(xMutexA);
    }
}

void TaskY(void *param) {
    for (;;) {
        xSemaphoreTake(xMutexA, portMAX_DELAY);  // Same order: A first
        xSemaphoreTake(xMutexB, portMAX_DELAY);  // Then B
        
        // Use both resources...
        
        xSemaphoreGive(xMutexB);
        xSemaphoreGive(xMutexA);
    }
}
```

---

## Recursive Mutexes

A **recursive mutex** can be taken multiple times by the same task:

```c
SemaphoreHandle_t xRecursiveMutex;

void init(void) {
    xRecursiveMutex = xSemaphoreCreateRecursiveMutex();
}

void OuterFunction(void) {
    xSemaphoreTakeRecursive(xRecursiveMutex, portMAX_DELAY);
    
    // Do something...
    InnerFunction();  // Also takes the mutex
    
    xSemaphoreGiveRecursive(xRecursiveMutex);
}

void InnerFunction(void) {
    xSemaphoreTakeRecursive(xRecursiveMutex, portMAX_DELAY);
    
    // Do something...
    
    xSemaphoreGiveRecursive(xRecursiveMutex);
}
```

---

## Lecture Materials

- [FreeRTOS Resource Management]({{ site.baseurl }}/ece3623/FreeRTOS.pdf)
- [Week 4 Slides]({{ site.baseurl }}/ece3623/Week%204.pdf)
- [Week 4 Recap]({{ site.baseurl }}/ece3623/Week%204-R.pdf)

---

## Reading Assignments

1. *Mastering the FreeRTOS Real Time Kernel*, Chapter 7: Resource Management
2. FreeRTOS Documentation: Mutexes and Binary Semaphores
3. FreeRTOS Documentation: Critical Sections

---

## Practice Questions

1. What is a race condition? Provide an example.
2. Explain the difference between `taskENTER_CRITICAL()` and using a mutex.
3. Why should critical sections be kept as short as possible?
4. What is deadlock? Describe one strategy to prevent it.
5. When would you use a recursive mutex instead of a regular mutex?
6. A task holds Mutex A and needs Mutex B. Another task holds Mutex B and needs Mutex A. What happens and how can this be prevented?
7. Why can't you call `vTaskDelay()` inside a critical section?

---

## Summary

This module covered mutual exclusion techniques for protecting shared resources in FreeRTOS. Critical sections (`taskENTER_CRITICAL/EXIT`) provide fast, lightweight protection by disabling interrupts, but must be kept short. Mutexes offer a more flexible solution for longer operations, allowing other tasks to run while one task holds the resource. Understanding these mechanisms and their trade-offs is essential for building correct, reliable multi-tasking systems.

---

## Next Module

[Module 5: Priority Inversion and Inheritance →](../module05/)                xil_printf("Worker suspended\r\n");
            }
        }
        vTaskDelay(pdMS_TO_TICKS(50));  /* Debounce */
    }
}
```

---

## Task Notifications

Lightweight alternative to semaphores for simple signaling.

```c
/* Send notification */
xTaskNotifyGive(xTaskHandle);

/* Wait for notification */
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
```

---

## Debugging Tasks

### Getting Task Information

```c
/* Get current task handle */
TaskHandle_t xHandle = xTaskGetCurrentTaskHandle();

/* Get task name */
char *pcTaskName = pcTaskGetName(xHandle);

/* Get task state */
eTaskState eState = eTaskGetState(xHandle);

/* Get stack high water mark (minimum free stack) */
UBaseType_t uxHighWaterMark = uxTaskGetStackHighWaterMark(xHandle);
```

### Runtime Statistics
Enable with `configGENERATE_RUN_TIME_STATS`.

---

## Lab Connection

### Lab 1: Task Management in FreeRTOS

In this lab, you will:
1. Create multiple tasks with different priorities
2. Observe preemption behavior
3. Implement periodic tasks using `vTaskDelayUntil()`
4. Monitor task execution with UART output

**Lab Materials**: [Lab 1 - Task Management]({{ site.baseurl }}/ece3623/LAB%201%20Task%20Management%20in%20FreeRTOS%20_2023Fall.pdf)

---

## Lecture Materials

- [Week 4 Slides]({{ site.baseurl }}/ece3623/Week%204-T.pdf)
- [Exam 1 Review Session]({{ site.baseurl }}/ece3623/Week%204-F_%20Review%20Session%20for%20Exam%201-1.pdf)

---

## Reading Assignments

1. FreeRTOS Kernel Documentation: Task Management
2. *Mastering FreeRTOS*, Chapter 3: Task Management
3. Review priority inheritance protocol

---

## Practice Questions

1. What are the four possible states of a FreeRTOS task?

2. A task with priority 3 is running. A task with priority 5 becomes ready. What happens?

3. Explain the difference between `vTaskDelay()` and `vTaskDelayUntil()`.

4. A task has a stack size of 512 words. How many bytes is this on a 32-bit system?

5. Write code to create a task that prints "Hello" every 250ms using `vTaskDelayUntil()`.

---

## Summary

Task management is fundamental to FreeRTOS applications. We covered task creation, states, priorities, delays, and control functions. Understanding these concepts enables you to build responsive, well-structured embedded applications. The next module introduces hardware timers and counters for precise timing control.

---

## Next Module

[Module 5: Counter-Timers →](../module05/)
