---
title: "Lab 4: Software Timers and Watchdog"
layout: default
parent: Laboratory Exercises
nav_order: 4
---

# Lab 4: Software Timers and Watchdog
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will implement FreeRTOS software timers for periodic operations and configure the Zynq hardware watchdog timer for system reliability. You will create timer callbacks, implement watchdog kicking strategies, and understand timing-critical operations.

**Related Modules**: [Module 8: Software Timers and Watchdog]({{ site.baseurl }}/docs/modules/module08.html), [Module 9: ADC and SPI]({{ site.baseurl }}/docs/modules/module09.html)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Create and manage FreeRTOS software timers
2. Implement one-shot and auto-reload timer modes
3. Configure the Zynq System Watchdog Timer (SWDT)
4. Design watchdog kicking strategies
5. Use timers for LED blinking and periodic measurements

---

## Prerequisites

- Completion of Labs 1-3
- Understanding of FreeRTOS tasks and queues
- Familiarity with callback functions

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- Micro-USB cable

---

## Background

### Software Timers vs Hardware Timers

| Aspect | Software Timer | Hardware Timer |
|--------|---------------|----------------|
| Resource | FreeRTOS kernel | Dedicated hardware |
| Precision | Tick-based (~10ms) | Nanosecond possible |
| Count | Many (memory limited) | Few (hardware limited) |
| Callback | Runs in timer daemon task | Runs in ISR |
| Use case | Application events | Precise timing, PWM |

### Timer Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOFTWARE TIMER STATES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   xTimerCreate()              xTimerStart()                     │
│        │                           │                             │
│        ▼                           ▼                             │
│   ┌─────────┐                 ┌─────────┐                       │
│   │ DORMANT │────────────────►│ RUNNING │◄──────────┐           │
│   └─────────┘   xTimerStart() └────┬────┘           │           │
│        ▲                           │                │           │
│        │ xTimerStop()              │ Timer expires  │           │
│        └───────────────────────────┤                │           │
│                                    ▼                │           │
│                          ┌──────────────────┐       │           │
│                          │ Callback Executes│       │           │
│                          └────────┬─────────┘       │           │
│                                   │                 │           │
│                     One-shot?  ───┼───  Auto-reload?│           │
│                         │         │            │    │           │
│                         ▼         │            └────┘           │
│                    Back to DORMANT│                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Basic Software Timer

### LED Blink Timer

```c
#include "FreeRTOS.h"
#include "task.h"
#include "timers.h"
#include "xgpio.h"
#include "xil_printf.h"

TimerHandle_t xBlinkTimer;
XGpio GpioLeds;
static uint8_t ledState = 0;

// Timer callback - called every 500ms
void vBlinkTimerCallback(TimerHandle_t xTimer) {
    ledState ^= 0x01;  // Toggle LED
    XGpio_DiscreteWrite(&GpioLeds, 1, ledState);
    xil_printf("LED: %s\r\n", ledState ? "ON" : "OFF");
}

int main(void) {
    // Initialize GPIO for LEDs
    XGpio_Initialize(&GpioLeds, XPAR_GPIO_0_DEVICE_ID);
    XGpio_SetDataDirection(&GpioLeds, 1, 0x00);  // Output
    
    xil_printf("Timer Demo Starting...\r\n");
    
    // Create auto-reload timer (500ms period)
    xBlinkTimer = xTimerCreate(
        "BlinkTimer",              // Name
        pdMS_TO_TICKS(500),        // Period
        pdTRUE,                    // Auto-reload = YES
        (void *)0,                 // Timer ID
        vBlinkTimerCallback        // Callback function
    );
    
    if (xBlinkTimer == NULL) {
        xil_printf("Error: Timer creation failed!\r\n");
        return -1;
    }
    
    // Start the timer
    if (xTimerStart(xBlinkTimer, 0) != pdPASS) {
        xil_printf("Error: Timer start failed!\r\n");
        return -1;
    }
    
    xil_printf("Timer started successfully\r\n");
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Part 2: One-Shot Timer (Timeout)

### Button Debounce Timer

```c
TimerHandle_t xDebounceTimer;
volatile uint8_t buttonProcessed = 0;

// One-shot timer for debounce
void vDebounceCallback(TimerHandle_t xTimer) {
    // Debounce period expired, allow next button press
    buttonProcessed = 0;
    xil_printf("Debounce complete - ready for next press\r\n");
}

void vButtonISR(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    if (!buttonProcessed) {
        buttonProcessed = 1;
        
        // Process the button
        u32 buttons = XGpio_DiscreteRead(&GpioButtons, 1);
        xil_printf("Button pressed: 0x%X\r\n", buttons);
        
        // Start debounce timer (50ms one-shot)
        xTimerStartFromISR(xDebounceTimer, &xHigherPriorityTaskWoken);
    }
    
    // Clear interrupt
    XGpio_InterruptClear(&GpioButtons, XGPIO_IR_CH1_MASK);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

int main(void) {
    // Create one-shot debounce timer (50ms)
    xDebounceTimer = xTimerCreate(
        "Debounce",
        pdMS_TO_TICKS(50),
        pdFALSE,               // One-shot = NO auto-reload
        (void *)0,
        vDebounceCallback
    );
    
    // ... rest of initialization
}
```

---

## Part 3: Multiple Timers

### Multi-Rate Sampling System

```c
TimerHandle_t xFastTimer;   // 100ms
TimerHandle_t xMediumTimer; // 500ms
TimerHandle_t xSlowTimer;   // 2000ms

void vFastTimerCallback(TimerHandle_t xTimer) {
    static int count = 0;
    xil_printf("FAST [%d]: Sampling sensors...\r\n", count++);
}

void vMediumTimerCallback(TimerHandle_t xTimer) {
    static int count = 0;
    xil_printf("MEDIUM [%d]: Updating display...\r\n", count++);
}

void vSlowTimerCallback(TimerHandle_t xTimer) {
    static int count = 0;
    xil_printf("SLOW [%d]: Logging to storage...\r\n", count++);
}

int main(void) {
    // Create timers at different rates
    xFastTimer = xTimerCreate("Fast", pdMS_TO_TICKS(100), pdTRUE, 
                               (void*)1, vFastTimerCallback);
    xMediumTimer = xTimerCreate("Medium", pdMS_TO_TICKS(500), pdTRUE, 
                                 (void*)2, vMediumTimerCallback);
    xSlowTimer = xTimerCreate("Slow", pdMS_TO_TICKS(2000), pdTRUE, 
                               (void*)3, vSlowTimerCallback);
    
    // Start all timers
    xTimerStart(xFastTimer, 0);
    xTimerStart(xMediumTimer, 0);
    xTimerStart(xSlowTimer, 0);
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Part 4: Watchdog Timer

### Zynq SWDT Configuration

```c
#include "xscuwdt.h"

XScuWdt WatchdogInstance;

int InitWatchdog(void) {
    XScuWdt_Config *WdtConfig;
    int Status;
    
    // Initialize watchdog
    WdtConfig = XScuWdt_LookupConfig(XPAR_SCUWDT_0_DEVICE_ID);
    Status = XScuWdt_CfgInitialize(&WatchdogInstance, WdtConfig,
                                    WdtConfig->BaseAddr);
    if (Status != XST_SUCCESS) return XST_FAILURE;
    
    // Load watchdog counter (timeout value)
    // Counter decrements at CPU_CLK/2
    // For ~2 second timeout at 650MHz: 650M/2 * 2 = 650,000,000
    XScuWdt_LoadWdt(&WatchdogInstance, 0x26BE3680);
    
    // Start watchdog in watchdog mode (reset on timeout)
    XScuWdt_SetWdMode(&WatchdogInstance);
    XScuWdt_Start(&WatchdogInstance);
    
    xil_printf("Watchdog started - must kick every 2 seconds!\r\n");
    
    return XST_SUCCESS;
}

void KickWatchdog(void) {
    XScuWdt_RestartWdt(&WatchdogInstance);
}
```

### Watchdog Kicking Task

```c
void vWatchdogTask(void *pvParameters) {
    InitWatchdog();
    
    for (;;) {
        // Kick the watchdog
        KickWatchdog();
        xil_printf("Watchdog kicked\r\n");
        
        // Wait before next kick (must be less than timeout!)
        vTaskDelay(pdMS_TO_TICKS(1000));  // Kick every 1 second
    }
}

// Simulated task that might hang
void vWorkerTask(void *pvParameters) {
    int iteration = 0;
    
    for (;;) {
        xil_printf("Worker: iteration %d\r\n", iteration++);
        
        // Simulate occasional long operation
        if (iteration == 10) {
            xil_printf("Worker: Simulating hang...\r\n");
            
            // This infinite loop will cause watchdog reset!
            while (1) {
                // Stuck! Watchdog will reset system.
            }
        }
        
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

## Part 5: Timer-Controlled Watchdog

### Smart Watchdog Pattern

```c
// Only kick watchdog if all tasks check in
#define NUM_TASKS  3

volatile uint8_t taskCheckins[NUM_TASKS] = {0};

void vTask1(void *pvParameters) {
    for (;;) {
        // Do work...
        xil_printf("Task 1 working\r\n");
        
        // Check in with watchdog monitor
        taskCheckins[0] = 1;
        
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

void vTask2(void *pvParameters) {
    for (;;) {
        // Do work...
        xil_printf("Task 2 working\r\n");
        taskCheckins[1] = 1;
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

void vTask3(void *pvParameters) {
    for (;;) {
        // Do work...
        xil_printf("Task 3 working\r\n");
        taskCheckins[2] = 1;
        vTaskDelay(pdMS_TO_TICKS(400));
    }
}

// Timer callback - checks if all tasks are healthy
void vWatchdogTimerCallback(TimerHandle_t xTimer) {
    int allHealthy = 1;
    
    // Check if all tasks have checked in
    for (int i = 0; i < NUM_TASKS; i++) {
        if (taskCheckins[i] == 0) {
            xil_printf("WARNING: Task %d did not check in!\r\n", i+1);
            allHealthy = 0;
        }
        taskCheckins[i] = 0;  // Reset for next period
    }
    
    if (allHealthy) {
        KickWatchdog();
        xil_printf("All tasks healthy - watchdog kicked\r\n");
    } else {
        xil_printf("FAULT: Not kicking watchdog - reset pending!\r\n");
    }
}
```

---

## Deliverables

1. **Source Code**:
   - LED blink timer implementation
   - Button debounce with one-shot timer
   - Watchdog implementation

2. **Demonstration**:
   - Show LED blinking at correct rate
   - Show watchdog reset when task hangs
   - Show recovery after watchdog reset

3. **Lab Report**:
   - Timer accuracy measurements
   - Watchdog timeout calculations
   - Analysis of kicking strategies

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Timer callback never runs | Check `configUSE_TIMERS` is 1 in FreeRTOSConfig.h |
| Timer period inaccurate | Check `configTICK_RATE_HZ` setting |
| Watchdog resets too fast | Increase counter load value |
| System resets immediately | Timer daemon priority may be too low |

---

## Reference Materials

- [Module 8: Software Timers and Watchdog]({{ site.baseurl }}/docs/modules/module08.html)
- [Introduction to Watchdog Timers]({{ site.baseurl }}/docs/pdfs/Introduction%20to%20Watchdog%20Timers.pdf)
- [FreeRTOS Timer API](https://www.freertos.org/FreeRTOS-Software-Timer-API-Functions.html)
