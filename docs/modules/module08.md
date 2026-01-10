---
title: "Module 8: Software Timers and Watchdog"
layout: default
parent: "Progression 3: Timing & Peripherals"
grand_parent: "Course Modules"
nav_order: 1
---

# Module 8: Software Timers and Watchdog
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Create and manage FreeRTOS software timers using `xTimerCreate()`
2. Distinguish between one-shot and auto-reload timer modes
3. Implement timer callback functions for periodic operations
4. Configure the Zynq hardware watchdog timer
5. Design watchdog kicking strategies for system reliability

---

## Overview

Software timers allow executing functions at specific times or periodically without dedicating a task. Watchdog timers provide a safety mechanism to recover from software failures. Together, these timing mechanisms are essential for reliable embedded systems.

---

## FreeRTOS Software Timers

### What are Software Timers?

Software timers are kernel-managed timers that execute a callback function when they expire:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOFTWARE TIMER LIFECYCLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   xTimerCreate()                xTimerStart()                   │
│        │                             │                           │
│        ▼                             ▼                           │
│   ┌─────────┐                   ┌─────────┐                     │
│   │ DORMANT │──────────────────►│ RUNNING │                     │
│   └─────────┘    xTimerStart()  └────┬────┘                     │
│        ▲                             │                           │
│        │         xTimerStop()        │  Timer expires            │
│        └─────────────────────────────┤                           │
│                                      ▼                           │
│                             ┌────────────────┐                   │
│                             │ Callback runs! │                   │
│                             └────────────────┘                   │
│                                      │                           │
│                   Auto-reload?  ─────┼─────  One-shot?          │
│                         │            │            │              │
│                         ▼            │            ▼              │
│                   Back to RUNNING    │       Back to DORMANT     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Callback context** | Runs in timer daemon task context |
| **No dedicated task** | Multiple timers share one task |
| **Non-blocking callbacks** | Callbacks must not block! |
| **Configurable** | Period, mode, and callback |

---

## Creating Software Timers

### xTimerCreate()

```c
#include "FreeRTOS.h"
#include "timers.h"

TimerHandle_t xTimerCreate(
    const char *pcTimerName,           // Name for debugging
    TickType_t xTimerPeriodInTicks,    // Timer period
    UBaseType_t uxAutoReload,          // pdTRUE = auto-reload, pdFALSE = one-shot
    void *pvTimerID,                   // ID passed to callback
    TimerCallbackFunction_t pxCallbackFunction  // Callback function
);
```

### Timer Callback Function Signature

```c
void vTimerCallback(TimerHandle_t xTimer);
```

### Example: Creating Timers

```c
#include "FreeRTOS.h"
#include "timers.h"
#include "xil_printf.h"

// Timer handles
TimerHandle_t xPeriodicTimer;
TimerHandle_t xOneShotTimer;

// Periodic timer callback
void vPeriodicCallback(TimerHandle_t xTimer) {
    static int count = 0;
    xil_printf("Periodic timer fired! Count: %d\r\n", ++count);
    
    // Toggle LED
    ToggleLED();
}

// One-shot timer callback
void vOneShotCallback(TimerHandle_t xTimer) {
    xil_printf("One-shot timer expired!\r\n");
    
    // Perform one-time action
    SendNotification();
}

int main(void) {
    // Create periodic timer: 1 second period, auto-reload
    xPeriodicTimer = xTimerCreate(
        "PeriodicTimer",                    // Name
        pdMS_TO_TICKS(1000),                // 1000ms period
        pdTRUE,                             // Auto-reload = YES
        (void *)0,                          // Timer ID
        vPeriodicCallback                   // Callback function
    );
    
    // Create one-shot timer: 5 second timeout
    xOneShotTimer = xTimerCreate(
        "OneShotTimer",
        pdMS_TO_TICKS(5000),                // 5000ms period
        pdFALSE,                            // Auto-reload = NO (one-shot)
        (void *)1,                          // Timer ID
        vOneShotCallback
    );
    
    if (xPeriodicTimer != NULL && xOneShotTimer != NULL) {
        // Start the timers
        xTimerStart(xPeriodicTimer, 0);
        xTimerStart(xOneShotTimer, 0);
    }
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Timer Control Functions

### Starting and Stopping Timers

```c
// Start a timer (from task context)
BaseType_t xTimerStart(TimerHandle_t xTimer, TickType_t xTicksToWait);

// Stop a timer
BaseType_t xTimerStop(TimerHandle_t xTimer, TickType_t xTicksToWait);

// Reset timer (restart countdown)
BaseType_t xTimerReset(TimerHandle_t xTimer, TickType_t xTicksToWait);

// Change timer period
BaseType_t xTimerChangePeriod(TimerHandle_t xTimer, 
                               TickType_t xNewPeriod,
                               TickType_t xTicksToWait);
```

### ISR-Safe Versions

```c
// Start from ISR
BaseType_t xTimerStartFromISR(TimerHandle_t xTimer,
                               BaseType_t *pxHigherPriorityTaskWoken);

// Reset from ISR
BaseType_t xTimerResetFromISR(TimerHandle_t xTimer,
                               BaseType_t *pxHigherPriorityTaskWoken);
```

### Example: Timeout Pattern

```c
TimerHandle_t xTimeoutTimer;

void vTimeoutCallback(TimerHandle_t xTimer) {
    xil_printf("ERROR: Operation timed out!\r\n");
    HandleTimeout();
}

void StartOperationWithTimeout(void) {
    // Start or reset the timeout timer
    xTimerReset(xTimeoutTimer, 0);
    
    // Begin operation...
    BeginNetworkTransfer();
}

void OperationCompleteHandler(void) {
    // Operation succeeded, stop the timeout timer
    xTimerStop(xTimeoutTimer, 0);
    
    xil_printf("Operation completed successfully\r\n");
}
```

---

## Timer Daemon Task

Software timers run in the context of a special **Timer Daemon Task** created by FreeRTOS:

### Configuration in FreeRTOSConfig.h

```c
// Enable software timers
#define configUSE_TIMERS                    1

// Timer daemon task priority (should be high)
#define configTIMER_TASK_PRIORITY           (configMAX_PRIORITIES - 1)

// Timer daemon task stack size
#define configTIMER_TASK_STACK_DEPTH        256

// Timer command queue length
#define configTIMER_QUEUE_LENGTH            10
```

### Important Considerations

| Aspect | Guideline |
|--------|-----------|
| **Callback duration** | Keep callbacks SHORT (don't block!) |
| **Priority** | Daemon task should have high priority |
| **Queue length** | Must accommodate all pending timer commands |
| **Shared context** | All timer callbacks run in same task |

### What NOT to Do in Timer Callbacks

```c
// BAD: This callback blocks!
void vBadCallback(TimerHandle_t xTimer) {
    // DON'T DO THIS:
    vTaskDelay(pdMS_TO_TICKS(100));              // Blocks!
    xSemaphoreTake(xMutex, portMAX_DELAY);       // Blocks!
    xQueueReceive(xQueue, &data, portMAX_DELAY); // Blocks!
}

// GOOD: Non-blocking callback
void vGoodCallback(TimerHandle_t xTimer) {
    // Signal a task to do the work
    xSemaphoreGive(xWorkSemaphore);
    
    // Or send to queue without blocking
    xQueueSend(xCommandQueue, &cmd, 0);
}
```

---

## Practical Timer Examples

### LED Blink Timer

```c
TimerHandle_t xBlinkTimer;
static int led_state = 0;

void vBlinkCallback(TimerHandle_t xTimer) {
    led_state = !led_state;
    SetLED(led_state);
}

void StartBlinking(int period_ms) {
    if (xBlinkTimer == NULL) {
        xBlinkTimer = xTimerCreate("Blink", pdMS_TO_TICKS(period_ms),
                                    pdTRUE, NULL, vBlinkCallback);
    }
    xTimerStart(xBlinkTimer, 0);
}

void StopBlinking(void) {
    xTimerStop(xBlinkTimer, 0);
}
```

### Debounce Timer

```c
TimerHandle_t xDebounceTimer;
volatile int button_stable = 0;

void vDebounceCallback(TimerHandle_t xTimer) {
    // Debounce period elapsed, read stable button state
    button_stable = ReadButton();
    
    if (button_stable) {
        xil_printf("Button confirmed pressed\r\n");
        HandleButtonPress();
    }
}

void ButtonISR(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // Clear interrupt
    ClearButtonInterrupt();
    
    // Reset debounce timer (50ms debounce period)
    xTimerResetFromISR(xDebounceTimer, &xHigherPriorityTaskWoken);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## Watchdog Timers

### Why Watchdog Timers?

A **watchdog timer (WDT)** is a hardware safety mechanism that resets the system if software fails to "kick" (refresh) it periodically:

```
┌─────────────────────────────────────────────────────────────────┐
│                    WATCHDOG OPERATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  NORMAL OPERATION (software healthy):                           │
│                                                                  │
│  ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐                │
│  │Kick│────│Kick│────│Kick│────│Kick│────│Kick│─── ...         │
│  └────┘    └────┘    └────┘    └────┘    └────┘                │
│   WDT      WDT       WDT       WDT       WDT                    │
│  resets   resets    resets    resets    resets                  │
│                                                                  │
│  FAILURE (software stuck/crashed):                              │
│                                                                  │
│  ┌────┐    ┌────┐                                               │
│  │Kick│────│Kick│───────────────── TIMEOUT! ──► SYSTEM RESET   │
│  └────┘    └────┘    ↑                                          │
│                      └─ No kick received!                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Use Cases

- **Recovering from crashes**: Software hangs or enters infinite loop
- **Deadlock detection**: Tasks blocked waiting for each other
- **Stack overflow**: Corruption causing erratic behavior
- **External interference**: EMI causing processor glitches

---

## Zynq Watchdog Timer (SCUWDT)

The Zynq PS includes a System Control Unit Watchdog Timer (SCUWDT).

### Features

| Feature | Value |
|---------|-------|
| Counter width | 32-bit |
| Clock source | CPU clock / 2 |
| Modes | Timer mode, Watchdog mode |
| Output | System reset or interrupt |

### Configuring the Watchdog

```c
#include "xscuwdt.h"

XScuWdt WdtInstance;

int Watchdog_Init(uint32_t timeout_ms) {
    XScuWdt_Config *ConfigPtr;
    int Status;
    
    // Look up configuration
    ConfigPtr = XScuWdt_LookupConfig(XPAR_SCUWDT_0_DEVICE_ID);
    if (ConfigPtr == NULL) {
        return XST_FAILURE;
    }
    
    // Initialize driver
    Status = XScuWdt_CfgInitialize(&WdtInstance, ConfigPtr,
                                    ConfigPtr->BaseAddr);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }
    
    // Calculate load value
    // WDT clock = CPU clock / 2
    uint32_t wdt_clock = XPAR_CPU_CORTEXA9_0_CPU_CLK_FREQ_HZ / 2;
    uint32_t load_value = (wdt_clock / 1000) * timeout_ms;
    
    // Load the timeout value
    XScuWdt_LoadWdt(&WdtInstance, load_value);
    
    // Set watchdog mode (reset on timeout)
    XScuWdt_SetTimerMode(&WdtInstance);
    
    return XST_SUCCESS;
}

void Watchdog_Start(void) {
    XScuWdt_Start(&WdtInstance);
}

void Watchdog_Kick(void) {
    XScuWdt_RestartWdt(&WdtInstance);
}

void Watchdog_Stop(void) {
    XScuWdt_Stop(&WdtInstance);
}
```

---

## Watchdog Kicking Strategies

### Strategy 1: Simple Main Loop Kick

```c
int main(void) {
    Watchdog_Init(1000);  // 1 second timeout
    Watchdog_Start();
    
    while (1) {
        ProcessSensors();
        UpdateActuators();
        CheckCommunication();
        
        // Kick at end of loop
        Watchdog_Kick();
    }
}
```

**Problem**: Only proves main loop runs, not that all subsystems work.

### Strategy 2: Multi-Task Health Monitoring (Recommended)

```c
// Health flags - each task sets its flag periodically
volatile uint8_t task1_alive = 0;
volatile uint8_t task2_alive = 0;
volatile uint8_t task3_alive = 0;

// Watchdog monitor task
void vWatchdogTask(void *param) {
    for (;;) {
        // Check all tasks reported healthy
        if (task1_alive && task2_alive && task3_alive) {
            // All systems go - kick watchdog
            Watchdog_Kick();
            
            // Reset flags for next period
            task1_alive = 0;
            task2_alive = 0;
            task3_alive = 0;
        } else {
            xil_printf("WARNING: Task health check failed!\r\n");
            // Don't kick - let watchdog reset the system
        }
        
        vTaskDelay(pdMS_TO_TICKS(500));  // Check every 500ms
    }
}

// Worker task 1
void vTask1(void *param) {
    for (;;) {
        // Do work...
        DoTask1Work();
        
        // Report healthy
        task1_alive = 1;
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### Strategy 3: Using a Software Timer

```c
TimerHandle_t xWatchdogKickTimer;
volatile uint8_t system_healthy = 1;

void vWatchdogKickCallback(TimerHandle_t xTimer) {
    if (system_healthy) {
        Watchdog_Kick();
        system_healthy = 0;  // Reset, tasks must set it again
    }
    // If not healthy, don't kick - system will reset
}

void init(void) {
    Watchdog_Init(2000);  // 2 second hardware timeout
    
    // Software timer kicks every 1 second (< hardware timeout)
    xWatchdogKickTimer = xTimerCreate("WDTKick", 
                                       pdMS_TO_TICKS(1000),
                                       pdTRUE,    // Auto-reload
                                       NULL,
                                       vWatchdogKickCallback);
    
    Watchdog_Start();
    xTimerStart(xWatchdogKickTimer, 0);
}
```

---

## Watchdog Best Practices

### Do's and Don'ts

| ✅ Do | ❌ Don't |
|------|---------|
| Use appropriate timeout (not too short) | Kick unconditionally in ISR |
| Monitor all critical tasks | Disable watchdog for debugging |
| Test recovery behavior | Rely on watchdog to mask bugs |
| Log reset reason | Ignore watchdog resets |

### Choosing Timeout Period

| System Type | Suggested Timeout |
|-------------|-------------------|
| Fast control loop | 100-500ms |
| Normal application | 1-2 seconds |
| Complex startup | 5-10 seconds |
| User interaction | 10-30 seconds |

### Debugging with Watchdog

```c
// Disable watchdog during debugging
#ifdef DEBUG_MODE
    #define WATCHDOG_ENABLED  0
#else
    #define WATCHDOG_ENABLED  1
#endif

void init_system(void) {
    #if WATCHDOG_ENABLED
        Watchdog_Init(2000);
        Watchdog_Start();
    #endif
}
```

---

## Lecture Materials

- [FreeRTOS Software Timers]({{ site.baseurl }}/ece3623/FreeRTOS.pdf)
- [Week 8 Slides]({{ site.baseurl }}/ece3623/Week%208.pdf)
- [Watchdog Timer Design]({{ site.baseurl }}/ece3623/Watchdog.pdf)

---

## Reading Assignments

1. *Mastering the FreeRTOS Real Time Kernel*, Chapter 5: Software Timer Management
2. Zynq-7000 TRM, Chapter 8: System Watchdog Timer
3. FreeRTOS Documentation: Software Timers API

---

## Practice Questions

1. What is the difference between a one-shot and an auto-reload software timer?
2. Why must software timer callbacks be non-blocking?
3. What is the purpose of the timer daemon task in FreeRTOS?
4. Explain why simply kicking the watchdog in the main loop is not sufficient for a multi-task system.
5. A watchdog has a 2-second timeout. How often should software kick it?
6. What happens if a timer callback takes too long to execute?
7. Describe a scenario where a watchdog timer would trigger a system reset.

---

## Summary

FreeRTOS software timers enable periodic or one-shot execution of callback functions without dedicated tasks. They run in the timer daemon task context and must not block. Watchdog timers provide hardware-level protection against software failures by resetting the system if not periodically kicked. Effective watchdog strategies monitor all critical system components, not just a single loop. Together, these timing mechanisms form the foundation for reliable, self-recovering embedded systems.

---

## Next Module

[Module 9: ADC and SPI Communication (PmodAD1) →]({{ site.baseurl }}/docs/modules/module09.html)

```c
/* Multiple tasks synchronize at a point */
#define TASK1_BIT  (1 << 0)
#define TASK2_BIT  (1 << 1)
#define TASK3_BIT  (1 << 2)
#define ALL_SYNC_BITS (TASK1_BIT | TASK2_BIT | TASK3_BIT)

void vTask1(void *p) {
    for(;;) {
        DoTask1Work();
        
        /* Wait for all tasks to reach this point */
        xEventGroupSync(xEventGroup, TASK1_BIT, ALL_SYNC_BITS, portMAX_DELAY);
        
        /* All tasks synchronized - continue */
    }
}
```

---

## Common Pitfalls

### Deadlock

Two tasks each hold a resource the other needs.

```c
/* Task 1 */               /* Task 2 */
Take(MutexA);              Take(MutexB);
Take(MutexB); /* Blocked */Take(MutexA); /* Blocked */
/* DEADLOCK! */
```

**Solution**: Always acquire mutexes in the same order.

### Starvation

A low-priority task never gets access to a resource.

**Solution**: Use priority inheritance, or time-slice access.

### Race Conditions

Access shared data without protection.

**Solution**: Always use mutexes or queues for shared data.

---

## Lab Connection

### Lab 3: FreeRTOS Data Queue on Zybo

In this lab, you will:
1. Create a data queue between producer and consumer tasks
2. Implement sensor data acquisition and processing
3. Use semaphores for synchronization
4. Observe queue behavior with different task priorities

**Lab Materials**: [Lab 3 - FreeRTOS Data Queue]({{ site.baseurl }}/ece3623/LAB%203%20FreeRTOS%20Data%20Queue%20on%20Zybo_2024FALL.pdf)

---

## Lecture Materials

- [Week 6 Slides]({{ site.baseurl }}/ece3623/Week%206-T.pdf)

---

## Reading Assignments

1. *Mastering FreeRTOS*, Chapter 4: Queue Management
2. *Mastering FreeRTOS*, Chapter 7: Resource Management
3. FreeRTOS documentation: Semaphores and Mutexes

---

## Practice Questions

1. What is the difference between a queue and a semaphore?

2. When would you use a counting semaphore instead of a binary semaphore?

3. Explain priority inheritance and why it's important.

4. Write code to create a queue of 5 structures, each containing an int and a float.

5. Two tasks need to access a shared UART. How would you protect this resource?

---

## Summary

Inter-task communication is essential for coordinated multi-task systems. Queues transfer data safely, semaphores provide synchronization, mutexes protect shared resources, and event groups handle complex multi-event scenarios. Proper use of these mechanisms prevents race conditions, deadlocks, and other concurrency issues. The next module covers serial communication protocols.

---

## Next Module

[Module 9: Serial Communication →]({{ site.baseurl }}/docs/modules/module09.html)
