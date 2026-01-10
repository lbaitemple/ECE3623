---
title: "Lab 2: Vivado AXI Interrupt"
layout: default
parent: Laboratory Exercises
nav_order: 2
---

# Lab 2: Vivado AXI Interrupt Controller
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will configure the AXI Interrupt Controller in Vivado and implement interrupt-driven programming on the Zybo Z7. You will connect GPIO buttons to generate interrupts and handle them using the Generic Interrupt Controller (GIC).

**Related Modules**: [Module 6: Hardware Interrupts and GIC]({{ site.baseurl }}/docs/modules/module06/), [Module 7: Queues]({{ site.baseurl }}/docs/modules/module07/)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Configure the AXI Interrupt Controller in Vivado
2. Connect GPIO peripherals to the interrupt system
3. Register and implement interrupt service routines (ISRs)
4. Use deferred interrupt handling with FreeRTOS

---

## Prerequisites

- Completion of Lab 1 and Modules 6-7
- Understanding of FreeRTOS task management
- Familiarity with Vivado block design

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- Micro-USB cable

---

## Background

### Interrupt System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZYNQ INTERRUPT SYSTEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│      ┌──────────────┐                                           │
│      │    Buttons   │──┐                                        │
│      │    (GPIO)    │  │                                        │
│      └──────────────┘  │                                        │
│                        │      ┌─────────────────┐               │
│      ┌──────────────┐  │      │                 │               │
│      │   Switches   │──┼─────►│  AXI Interrupt  │               │
│      │    (GPIO)    │  │      │   Controller    │               │
│      └──────────────┘  │      │                 │               │
│                        │      └────────┬────────┘               │
│      ┌──────────────┐  │               │                        │
│      │  AXI Timer   │──┘               │ IRQ_F2P                │
│      └──────────────┘                  │                        │
│                                        ▼                        │
│                        ┌───────────────────────────┐            │
│                        │          GIC              │            │
│                        │  (Generic Interrupt       │            │
│                        │   Controller)             │            │
│                        └───────────────┬───────────┘            │
│                                        │                        │
│                                        ▼                        │
│                              ┌──────────────────┐               │
│                              │  ARM Cortex-A9   │               │
│                              │     (CPU)        │               │
│                              └──────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![AXI Interrupt Architecture]({{ site.baseurl }}/docs/imgs/labs/lab2-img-000.png)

---

## Part 1: Vivado Hardware Design

### Step 1: Create Block Design

1. Create a new Vivado project for Zybo Z7
2. Add ZYNQ7 Processing System
3. Run Block Automation

### Step 2: Add GPIO Peripherals

1. Add **AXI GPIO** IP block
2. Configure for:
   - GPIO: 4-bit input (buttons)
   - GPIO2: 4-bit input (switches)
3. Enable interrupts in GPIO settings

### Step 3: Add AXI Interrupt Controller

1. Add **AXI Interrupt Controller** IP
2. Connect GPIO interrupt outputs to interrupt controller inputs
3. Connect interrupt controller output to `IRQ_F2P[0:0]`

### Step 4: Configure ZYNQ PS

1. Double-click ZYNQ PS
2. Navigate to **Interrupts** tab
3. Enable **IRQ_F2P[15:0]**

![Block Design with Interrupts]({{ site.baseurl }}/docs/imgs/labs/lab2-img-002.png)

### Step 5: Complete Connections

```
GPIO Buttons ──► GPIO IP ──► AXI INTC ──► IRQ_F2P[0] ──► GIC ──► CPU
```

1. Run Connection Automation
2. Validate design
3. Create HDL wrapper
4. Generate bitstream

---

## Part 2: Interrupt Handler Implementation

### Basic Interrupt Setup

```c
#include "xscugic.h"
#include "xgpio.h"
#include "xil_exception.h"
#include "xil_printf.h"

// Global instances
XScuGic InterruptController;
XGpio GpioButtons;

// Interrupt handler for GPIO
void GPIO_InterruptHandler(void *CallbackRef) {
    XGpio *GpioPtr = (XGpio *)CallbackRef;
    
    // Disable GPIO interrupt
    XGpio_InterruptDisable(GpioPtr, XGPIO_IR_CH1_MASK);
    
    // Clear the interrupt
    XGpio_InterruptClear(GpioPtr, XGPIO_IR_CH1_MASK);
    
    // Read button state
    u32 buttons = XGpio_DiscreteRead(GpioPtr, 1);
    xil_printf("Button pressed: 0x%X\r\n", buttons);
    
    // Re-enable GPIO interrupt
    XGpio_InterruptEnable(GpioPtr, XGPIO_IR_CH1_MASK);
}

int SetupInterruptSystem(void) {
    int Status;
    XScuGic_Config *GicConfig;
    
    // Initialize GIC
    GicConfig = XScuGic_LookupConfig(XPAR_SCUGIC_SINGLE_DEVICE_ID);
    Status = XScuGic_CfgInitialize(&InterruptController, GicConfig,
                                    GicConfig->CpuBaseAddress);
    if (Status != XST_SUCCESS) return XST_FAILURE;
    
    // Connect to ARM exception system
    Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                    (Xil_ExceptionHandler)XScuGic_InterruptHandler,
                    &InterruptController);
    Xil_ExceptionEnable();
    
    // Connect GPIO interrupt handler
    Status = XScuGic_Connect(&InterruptController,
                              XPAR_FABRIC_GPIO_0_VEC_ID,
                              (Xil_ExceptionHandler)GPIO_InterruptHandler,
                              &GpioButtons);
    if (Status != XST_SUCCESS) return XST_FAILURE;
    
    // Enable GPIO interrupt in GIC
    XScuGic_Enable(&InterruptController, XPAR_FABRIC_GPIO_0_VEC_ID);
    
    // Enable GPIO interrupts
    XGpio_InterruptEnable(&GpioButtons, XGPIO_IR_CH1_MASK);
    XGpio_InterruptGlobalEnable(&GpioButtons);
    
    return XST_SUCCESS;
}
```

---

## Part 3: FreeRTOS Integration

### Deferred Interrupt Handling

Instead of processing in the ISR, defer to a task:

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"

SemaphoreHandle_t xButtonSemaphore;

// ISR - minimal work, give semaphore
void GPIO_InterruptHandler(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    XGpio_InterruptDisable(&GpioButtons, XGPIO_IR_CH1_MASK);
    XGpio_InterruptClear(&GpioButtons, XGPIO_IR_CH1_MASK);
    
    // Signal the task
    xSemaphoreGiveFromISR(xButtonSemaphore, &xHigherPriorityTaskWoken);
    
    XGpio_InterruptEnable(&GpioButtons, XGPIO_IR_CH1_MASK);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// Handler Task - does actual processing
void vButtonHandlerTask(void *pvParameters) {
    for (;;) {
        // Wait for interrupt
        if (xSemaphoreTake(xButtonSemaphore, portMAX_DELAY) == pdTRUE) {
            // Read and process button state
            u32 buttons = XGpio_DiscreteRead(&GpioButtons, 1);
            
            xil_printf("Button Handler: Processing button 0x%X\r\n", buttons);
            
            // Do actual work here (can take time)
            switch (buttons) {
                case 0x01: xil_printf("BTN0 - Action 1\r\n"); break;
                case 0x02: xil_printf("BTN1 - Action 2\r\n"); break;
                case 0x04: xil_printf("BTN2 - Action 3\r\n"); break;
                case 0x08: xil_printf("BTN3 - Action 4\r\n"); break;
            }
        }
    }
}

int main(void) {
    // Initialize GPIO
    XGpio_Initialize(&GpioButtons, XPAR_GPIO_0_DEVICE_ID);
    XGpio_SetDataDirection(&GpioButtons, 1, 0xFFFFFFFF);  // Input
    
    // Create semaphore
    xButtonSemaphore = xSemaphoreCreateBinary();
    
    // Setup interrupts
    SetupInterruptSystem();
    
    // Create handler task
    xTaskCreate(vButtonHandlerTask, "BtnHandler", 2048, NULL, 3, NULL);
    
    // Start scheduler
    vTaskStartScheduler();
    
    for (;;);
}
```

---

## Part 4: Multiple Interrupt Sources

### Handling Buttons and Switches

```c
void vButtonTask(void *pvParameters) {
    for (;;) {
        if (xSemaphoreTake(xButtonSemaphore, portMAX_DELAY)) {
            u32 btn = XGpio_DiscreteRead(&GpioInstance, 1);
            xil_printf("Button: 0x%X\r\n", btn);
        }
    }
}

void vSwitchTask(void *pvParameters) {
    for (;;) {
        if (xSemaphoreTake(xSwitchSemaphore, portMAX_DELAY)) {
            u32 sw = XGpio_DiscreteRead(&GpioInstance, 2);
            xil_printf("Switch: 0x%X\r\n", sw);
        }
    }
}
```

---

## Deliverables

1. **Vivado Project**:
   - Block design with AXI Interrupt Controller
   - Bitstream file

2. **Source Code**:
   - Interrupt handler implementation
   - FreeRTOS deferred handling version

3. **Lab Report**:
   - Block design screenshot
   - Serial output showing interrupt handling
   - Analysis of interrupt latency

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Interrupt not firing | Check IRQ_F2P connection in block design |
| Handler not called | Verify GIC setup and interrupt enable |
| System hangs in ISR | Don't call blocking functions in ISR |
| Wrong interrupt ID | Check xparameters.h for correct IDs |

---

## Reference Materials

- [Lab 2 PDF Writeup](../pdfs/LAB%202%20Vivado%20AXI%20Interrupt%20.pdf)
- [Module 6: Hardware Interrupts]({{ site.baseurl }}/docs/modules/module06/)
- [Xilinx Interrupt Controller Documentation](https://www.xilinx.com/support/documentation/ip_documentation/axi_intc/v4_1/pg099-axi-intc.pdf)
