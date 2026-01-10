---
title: "Module 6: Hardware Interrupts and GIC"
layout: default
parent: Course Modules
nav_order: 6
---

# Module 6: Hardware Interrupts and the Generic Interrupt Controller
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the role of interrupts in embedded systems
2. Describe the Zynq Generic Interrupt Controller (GIC) architecture
3. Configure and register interrupt handlers in Xilinx Vitis
4. Design efficient Interrupt Service Routines (ISRs)
5. Integrate hardware interrupts with FreeRTOS using deferred interrupt handling

---

## Overview

Interrupts are fundamental to responsive embedded systems. Rather than continuously polling for events (wasting CPU cycles), interrupts allow the processor to respond immediately when hardware events occur. The Zynq-7000 uses the ARM Generic Interrupt Controller (GIC) to manage interrupts from both the Processing System and Programmable Logic.

---

## Interrupt Fundamentals

### Polling vs. Interrupt-Driven

| Approach | Method | Pros | Cons |
|----------|--------|------|------|
| **Polling** | Continuously check status | Simple, deterministic | Wastes CPU, slow response |
| **Interrupt** | Hardware signals CPU | Efficient, fast response | More complex, non-deterministic |

### Polling Example (Inefficient)

```c
// CPU constantly checks button state
while (1) {
    if (GPIO_ReadPin(BUTTON_PIN) == PRESSED) {
        HandleButtonPress();
    }
    // CPU is BUSY even when nothing happens!
}
```

### Interrupt Example (Efficient)

```c
// CPU does other work, responds only when button pressed
void ButtonISR(void *CallbackRef) {
    HandleButtonPress();
}

int main(void) {
    RegisterInterrupt(BUTTON_IRQ, ButtonISR);
    EnableInterrupt(BUTTON_IRQ);
    
    while (1) {
        DoOtherWork();  // CPU is productive
        // Or enter low-power sleep mode
    }
}
```

---

## Interrupt Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERRUPT LIFECYCLE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. TRIGGER     Hardware event occurs (timer, GPIO, UART, etc.) │
│       │                                                          │
│       ▼                                                          │
│  2. SIGNAL      Peripheral asserts interrupt line to GIC        │
│       │                                                          │
│       ▼                                                          │
│  3. PRIORITIZE  GIC determines if interrupt should be taken     │
│       │         (based on priority and current context)         │
│       ▼                                                          │
│  4. VECTOR      CPU jumps to interrupt vector table             │
│       │                                                          │
│       ▼                                                          │
│  5. SAVE        Context saved (registers pushed to stack)       │
│       │                                                          │
│       ▼                                                          │
│  6. EXECUTE     ISR handler runs                                │
│       │                                                          │
│       ▼                                                          │
│  7. CLEAR       ISR acknowledges/clears interrupt source        │
│       │                                                          │
│       ▼                                                          │
│  8. RESTORE     Context restored, return to interrupted code    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Zynq Generic Interrupt Controller (GIC)

The ARM GIC (PL390) in Zynq manages all interrupts for the dual Cortex-A9 cores.

### GIC Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 ZYNQ INTERRUPT SYSTEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │   Private   │    │   Shared    │    │     PL      │          │
│  │ Peripherals │    │ Peripherals │    │  Interrupts │          │
│  │ (per CPU)   │    │ (PS)        │    │ (FPGA)      │          │
│  │             │    │             │    │             │          │
│  │ - Timer     │    │ - UART      │    │ - Custom IP │          │
│  │ - Watchdog  │    │ - SPI       │    │ - AXI Timer │          │
│  │             │    │ - I2C       │    │ - GPIO      │          │
│  │             │    │ - GPIO      │    │             │          │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘          │
│         │                  │                  │                  │
│         │    IRQ IDs       │    IRQ IDs       │    IRQ IDs      │
│         │    0-31          │    32-95         │    61-68, 84-91 │
│         │   (PPI/SGI)      │    (SPI)         │    (SPI)        │
│         │                  │                  │                  │
│         └──────────────────┼──────────────────┘                  │
│                            ▼                                     │
│              ┌─────────────────────────────┐                    │
│              │   GIC Distributor           │                    │
│              │   - Enable/Disable IRQs     │                    │
│              │   - Priority assignment     │                    │
│              │   - CPU targeting           │                    │
│              └─────────────┬───────────────┘                    │
│                            │                                     │
│              ┌─────────────┴───────────────┐                    │
│              │                             │                     │
│              ▼                             ▼                     │
│    ┌──────────────────┐         ┌──────────────────┐            │
│    │ CPU Interface 0  │         │ CPU Interface 1  │            │
│    │ (CPU 0)          │         │ (CPU 1)          │            │
│    └────────┬─────────┘         └────────┬─────────┘            │
│             │                            │                       │
│             ▼                            ▼                       │
│        ARM Cortex-A9               ARM Cortex-A9                │
│           CPU 0                       CPU 1                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Interrupt Types

| Type | ID Range | Description |
|------|----------|-------------|
| **SGI** | 0-15 | Software Generated Interrupts (inter-processor) |
| **PPI** | 16-31 | Private Peripheral Interrupts (per CPU) |
| **SPI** | 32-95 | Shared Peripheral Interrupts (global) |

### Key Zynq Interrupt IDs

| Interrupt Source | IRQ ID | Type |
|------------------|--------|------|
| Private Timer | 29 | PPI |
| Watchdog Timer | 30 | PPI |
| TTC 0-0 | 42 | SPI |
| TTC 0-1 | 43 | SPI |
| TTC 0-2 | 44 | SPI |
| UART 0 | 59 | SPI |
| UART 1 | 82 | SPI |
| GPIO | 52 | SPI |
| SPI 0 | 58 | SPI |
| SPI 1 | 81 | SPI |
| **PL Interrupts** | 61-68, 84-91 | SPI |

---

## Configuring Interrupts in Vitis

### Step 1: Initialize the GIC

```c
#include "xscugic.h"
#include "xil_exception.h"

XScuGic InterruptController;  // GIC instance

int SetupInterruptSystem(void) {
    XScuGic_Config *GicConfig;
    int Status;
    
    // Look up GIC configuration
    GicConfig = XScuGic_LookupConfig(XPAR_SCUGIC_SINGLE_DEVICE_ID);
    if (GicConfig == NULL) {
        return XST_FAILURE;
    }
    
    // Initialize GIC driver
    Status = XScuGic_CfgInitialize(&InterruptController, 
                                    GicConfig,
                                    GicConfig->CpuBaseAddress);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }
    
    // Connect to ARM exception system
    Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                                 (Xil_ExceptionHandler)XScuGic_InterruptHandler,
                                 &InterruptController);
    
    // Enable interrupts in the ARM processor
    Xil_ExceptionEnable();
    
    return XST_SUCCESS;
}
```

### Step 2: Register an Interrupt Handler

```c
#include "xgpio.h"

XGpio Gpio;
volatile int ButtonPressed = 0;

// GPIO Interrupt Handler
void GpioHandler(void *CallbackRef) {
    XGpio *GpioPtr = (XGpio *)CallbackRef;
    
    // Disable GPIO interrupt to prevent re-entry
    XGpio_InterruptDisable(GpioPtr, XGPIO_IR_CH1_MASK);
    
    // Clear the interrupt
    XGpio_InterruptClear(GpioPtr, XGPIO_IR_CH1_MASK);
    
    // Set flag (minimal work in ISR)
    ButtonPressed = 1;
    
    // Re-enable GPIO interrupt
    XGpio_InterruptEnable(GpioPtr, XGPIO_IR_CH1_MASK);
}

int SetupGpioInterrupt(void) {
    int Status;
    
    // Initialize GPIO
    Status = XGpio_Initialize(&Gpio, XPAR_AXI_GPIO_0_DEVICE_ID);
    XGpio_SetDataDirection(&Gpio, 1, 0xFFFFFFFF);  // All inputs
    
    // Connect GPIO interrupt handler to GIC
    Status = XScuGic_Connect(&InterruptController,
                             XPAR_FABRIC_AXI_GPIO_0_IP2INTC_IRPT_INTR,
                             (Xil_InterruptHandler)GpioHandler,
                             (void *)&Gpio);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }
    
    // Enable interrupt in GIC
    XScuGic_Enable(&InterruptController,
                   XPAR_FABRIC_AXI_GPIO_0_IP2INTC_IRPT_INTR);
    
    // Enable GPIO interrupts
    XGpio_InterruptEnable(&Gpio, XGPIO_IR_CH1_MASK);
    XGpio_InterruptGlobalEnable(&Gpio);
    
    return XST_SUCCESS;
}
```

### Step 3: Set Interrupt Priority (Optional)

```c
// Set interrupt priority (0 = highest, 248 = lowest)
// Priority must be multiple of 8 for GIC
XScuGic_SetPriorityTriggerType(&InterruptController,
                                UART_INTERRUPT_ID,
                                0xA0,           // Priority (0-248)
                                0x03);          // Trigger: rising edge
```

---

## ISR Design Best Practices

### The Golden Rules

```
┌─────────────────────────────────────────────────────────────────┐
│                    ISR DESIGN RULES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. KEEP IT SHORT                                                │
│     • Minimize time in ISR (μs, not ms!)                        │
│     • Defer processing to tasks                                  │
│                                                                  │
│  2. NO BLOCKING                                                  │
│     • Never call blocking functions                              │
│     • No vTaskDelay(), no waiting for mutexes                   │
│                                                                  │
│  3. USE ISR-SAFE APIs                                            │
│     • FreeRTOS: Use "FromISR" versions only                     │
│     • xQueueSendFromISR(), xSemaphoreGiveFromISR()              │
│                                                                  │
│  4. CLEAR THE INTERRUPT                                          │
│     • Acknowledge interrupt source                               │
│     • Prevent infinite re-triggering                             │
│                                                                  │
│  5. MINIMIZE SHARED DATA ACCESS                                  │
│     • Use volatile for shared variables                          │
│     • Keep critical sections atomic                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Good ISR Example

```c
// ISR-safe semaphore for deferred handling
SemaphoreHandle_t xButtonSemaphore;

void GpioHandler(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // Clear interrupt source FIRST
    XGpio_InterruptClear(&Gpio, XGPIO_IR_CH1_MASK);
    
    // Signal task to handle button (very fast!)
    xSemaphoreGiveFromISR(xButtonSemaphore, &xHigherPriorityTaskWoken);
    
    // Yield if a higher priority task was woken
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### Bad ISR Example (Don't Do This!)

```c
// BAD: This ISR does too much!
void BadGpioHandler(void *CallbackRef) {
    // BAD: Printing in ISR
    xil_printf("Button pressed!\r\n");  
    
    // BAD: Complex calculation
    for (int i = 0; i < 1000; i++) {
        result += complexCalculation(i);
    }
    
    // BAD: Blocking call
    vTaskDelay(pdMS_TO_TICKS(100));  // NEVER DO THIS!
    
    // BAD: Non-ISR-safe FreeRTOS API
    xQueueSend(queue, &data, portMAX_DELAY);  // WRONG!
}
```

---

## Deferred Interrupt Handling with FreeRTOS

### The Pattern

ISR does minimal work, signals a task to do the real processing:

```
┌───────────┐          ┌───────────────┐          ┌───────────────┐
│ Interrupt │   Give   │  Semaphore/   │   Take   │  Handler      │
│    ISR    │─────────►│    Queue      │─────────►│    Task       │
│  (short)  │          │               │          │  (long work)  │
└───────────┘          └───────────────┘          └───────────────┘
     │                                                    │
     │  • Clear interrupt                                 │
     │  • Signal semaphore                                │
     │  • Return quickly                                  │
     │                                                    │
                                                          │
                                              • Process data        
                                              • Update display      
                                              • Send messages       
                                              • Complex logic       
```

### Complete Deferred Handling Example

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"
#include "xscugic.h"
#include "xgpio.h"

// Global handles
XScuGic Gic;
XGpio Gpio;
SemaphoreHandle_t xButtonSemaphore;
TaskHandle_t xButtonTaskHandle;

// ISR - runs in interrupt context
void GpioISR(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // 1. Clear the interrupt FIRST
    XGpio_InterruptClear(&Gpio, XGPIO_IR_CH1_MASK);
    
    // 2. Signal the handler task
    xSemaphoreGiveFromISR(xButtonSemaphore, &xHigherPriorityTaskWoken);
    
    // 3. Request context switch if needed
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// Handler Task - runs in task context
void ButtonHandlerTask(void *pvParameters) {
    for (;;) {
        // Block until ISR signals
        if (xSemaphoreTake(xButtonSemaphore, portMAX_DELAY) == pdTRUE) {
            // Safe to do complex processing here
            xil_printf("Button pressed! Processing...\r\n");
            
            // Debounce delay
            vTaskDelay(pdMS_TO_TICKS(50));
            
            // Read button state
            u32 buttonState = XGpio_DiscreteRead(&Gpio, 1);
            
            // Complex processing
            ProcessButtonEvent(buttonState);
            
            // Update display
            UpdateDisplay();
        }
    }
}

int main(void) {
    // Create binary semaphore
    xButtonSemaphore = xSemaphoreCreateBinary();
    
    // Create handler task (high priority for responsiveness)
    xTaskCreate(ButtonHandlerTask, "BtnHandler", 512, NULL, 
                configMAX_PRIORITIES - 1, &xButtonTaskHandle);
    
    // Initialize interrupt system
    SetupInterruptSystem();
    SetupGpioInterrupt();
    
    // Start scheduler
    vTaskStartScheduler();
    
    for (;;);
}
```

---

## FreeRTOS ISR-Safe API

| Regular API | ISR-Safe Version | Notes |
|-------------|------------------|-------|
| `xQueueSend()` | `xQueueSendFromISR()` | Must check `pxHigherPriorityTaskWoken` |
| `xQueueReceive()` | `xQueueReceiveFromISR()` | Use sparingly in ISR |
| `xSemaphoreGive()` | `xSemaphoreGiveFromISR()` | For binary/counting semaphores |
| `xSemaphoreTake()` | `xSemaphoreTakeFromISR()` | Use sparingly |
| `xTaskNotifyGive()` | `vTaskNotifyGiveFromISR()` | Lightweight signaling |
| N/A | `portYIELD_FROM_ISR()` | Request context switch |

### Important: Context Switch from ISR

```c
void MyISR(void *param) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // ... do ISR work ...
    
    xSemaphoreGiveFromISR(sem, &xHigherPriorityTaskWoken);
    
    // CRITICAL: Check if we should yield
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## Interrupt Priority and FreeRTOS

### Configuring Interrupt Priorities

FreeRTOS requires interrupts that call FreeRTOS API to have priority ≤ `configMAX_SYSCALL_INTERRUPT_PRIORITY`:

```c
// FreeRTOSConfig.h
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    (5 << 3)  // Priority 5

// Interrupts at priority 0-4: Cannot use FreeRTOS API (too high priority)
// Interrupts at priority 5-31: Can use FreeRTOS FromISR APIs
```

### Priority Mapping

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERRUPT PRIORITIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Priority 0 (Highest) ─────── Cannot use FreeRTOS API           │
│  Priority 1            ─────── Cannot use FreeRTOS API          │
│  Priority 2            ─────── Cannot use FreeRTOS API          │
│  Priority 3            ─────── Cannot use FreeRTOS API          │
│  Priority 4            ─────── Cannot use FreeRTOS API          │
│  ─────────────────────────────────────────────────────────────  │
│  Priority 5            ─────── configMAX_SYSCALL_INTERRUPT_PRIORITY
│  Priority 6            ─────── CAN use FreeRTOS FromISR APIs    │
│  Priority 7            ─────── CAN use FreeRTOS FromISR APIs    │
│  ...                                                             │
│  Priority 31 (Lowest)  ─────── CAN use FreeRTOS FromISR APIs    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lecture Materials

- [Interrupts and GIC]({{ site.baseurl }}/ece3623/Interrupts.pdf)
- [Week 6 Slides]({{ site.baseurl }}/ece3623/Week%206.pdf)
- [Week 6 Recap]({{ site.baseurl }}/ece3623/Week%206-R.pdf)

---

## Reading Assignments

1. Zynq-7000 TRM, Chapter 7: Interrupts
2. ARM Generic Interrupt Controller Architecture Specification
3. *Mastering the FreeRTOS Real Time Kernel*, Chapter 6: Interrupt Management

---

## Practice Questions

1. What are the advantages of interrupt-driven I/O over polling?
2. What is the GIC and what is its role in the Zynq interrupt system?
3. What is the difference between PPI and SPI interrupts?
4. List three rules for writing efficient ISRs.
5. Why must you use `xSemaphoreGiveFromISR()` instead of `xSemaphoreGive()` in an ISR?
6. What is deferred interrupt handling and why is it important?
7. What happens if an ISR that uses FreeRTOS APIs has priority higher than `configMAX_SYSCALL_INTERRUPT_PRIORITY`?

---

## Summary

This module covered hardware interrupts and the Zynq Generic Interrupt Controller (GIC). We learned how to configure the GIC, register interrupt handlers, and design efficient ISRs. The key principle is **deferred interrupt handling**: ISRs should be short and fast, signaling tasks to perform complex processing. FreeRTOS provides ISR-safe APIs for communication between interrupt and task contexts. Proper interrupt priority configuration ensures that FreeRTOS internals remain protected while allowing responsive interrupt handling.

---

## Next Module

[Module 7: Queues and Inter-Task Communication →](../module07/)
            task3_alive = 0;
        }
        /* If any flag not set, don't kick - system will reset */
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* In each monitored task */
void vTask1(void *pvParameters)
{
    extern volatile u8 task1_alive;
    
    for(;;) {
        /* Do work */
        DoTask1Work();
        
        /* Signal alive */
        task1_alive = 1;
        
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

### Strategy 3: Window Watchdog

Only accept kicks within a time window to detect tasks running too fast or too slow.

---

## Best Practices

### Watchdog Design Guidelines

1. **Set appropriate timeout**: Long enough for worst-case normal operation, short enough to detect failures quickly

2. **Don't kick too early**: Kicking in the wrong place can mask problems

3. **Monitor all critical tasks**: Use a supervisor task pattern

4. **Log before reset**: If possible, save diagnostic info before allowing reset

5. **Test the watchdog**: Intentionally let it expire during development

---

## Lecture Materials

- [Introduction to Timer Based Measurement]({{ site.baseurl }}/ece3623/Introduction%20to%20Timer%20Based%20Measurement.pdf)
- [Introduction to Watchdog Timers]({{ site.baseurl }}/ece3623/Introduction%20to%20Watchdog%20Timers.pdf)
- [Week 6 Slides]({{ site.baseurl }}/ece3623/Week%206-T.pdf)

---

## Reading Assignments

1. Zynq-7000 TRM, Chapter 9: System Watchdog Timer
2. Application notes on watchdog best practices
3. FreeRTOS documentation on idle task hooks (for watchdog integration)

---

## Practice Questions

1. A signal has a period of 2ms measured using a timer with 1µs resolution. What is its frequency?

2. Explain how to measure pulse width using input capture.

3. What happens if software fails to kick the watchdog timer?

4. Why is the "watchdog task" pattern better than kicking from multiple locations?

5. A watchdog has a 500ms timeout. Your main loop takes 100ms worst-case. Is this safe? What margin should you have?

---

## Summary

Timer-based measurements enable precise timing analysis of external signals and code execution. Watchdog timers provide essential protection against software failures, ensuring system reliability. Combining these techniques results in robust embedded systems. The next module covers interrupt handling for responsive event-driven programming.

---

## Next Module

[Module 7: Interrupts →](../module07/)
