---
title: "Lab 3: FreeRTOS Data Queue"
layout: default
parent: Laboratory Exercises
nav_order: 3
---

# Lab 3: FreeRTOS Data Queue on Zybo
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will implement producer-consumer patterns using FreeRTOS queues. You will create tasks that communicate through queues, transferring data safely between producers and consumers without shared variable issues.

**Related Modules**: [Module 7: Queues and Inter-Task Communication]({{ site.baseurl }}/docs/modules/module07.html), [Module 8: Software Timers]({{ site.baseurl }}/docs/modules/module08.html)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Create and configure FreeRTOS queues using `xQueueCreate()`
2. Send data to queues using `xQueueSend()` and variants
3. Receive data from queues using `xQueueReceive()`
4. Implement producer-consumer design patterns
5. Handle queue full/empty conditions with timeouts

---

## Prerequisites

- Completion of Labs 1-2
- Understanding of FreeRTOS tasks and interrupts
- Familiarity with C structures

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- Micro-USB cable

---

## Background

### Why Use Queues?

```
┌─────────────────────────────────────────────────────────────────┐
│              SHARED VARIABLE PROBLEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WITHOUT QUEUE:                   WITH QUEUE:                   │
│                                                                  │
│  Task A ──┐                       Task A ──┐                    │
│           │   shared_var           ┌──────┴──────┐              │
│  Task B ──┼────────────           │   QUEUE     │              │
│           │    RACE                │  (thread-   │──► Task C   │
│  Task C ──┘  CONDITION!            │   safe)     │              │
│                                    └─────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![Queue Data Flow]({{ site.baseurl }}/docs/imgs/labs/lab3-img-000.png)

### Queue Operations

| Function | Description |
|----------|-------------|
| `xQueueCreate()` | Create a new queue |
| `xQueueSend()` | Add item to back of queue |
| `xQueueSendToFront()` | Add item to front of queue |
| `xQueueReceive()` | Remove item from front |
| `xQueuePeek()` | Read without removing |
| `uxQueueMessagesWaiting()` | Check queue depth |

---

## Part 1: Basic Queue Operation

### Simple Integer Queue

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "xil_printf.h"

QueueHandle_t xIntegerQueue;

// Producer Task - generates integers
void vProducerTask(void *pvParameters) {
    int value = 0;
    
    for (;;) {
        xil_printf("Producer: Sending %d\r\n", value);
        
        if (xQueueSend(xIntegerQueue, &value, pdMS_TO_TICKS(100)) != pdPASS) {
            xil_printf("Producer: Queue full!\r\n");
        }
        
        value++;
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

// Consumer Task - processes integers
void vConsumerTask(void *pvParameters) {
    int received;
    
    for (;;) {
        if (xQueueReceive(xIntegerQueue, &received, portMAX_DELAY) == pdPASS) {
            xil_printf("Consumer: Received %d\r\n", received);
        }
    }
}

int main(void) {
    xil_printf("Queue Demo Starting...\r\n");
    
    // Create queue: 10 items, each sizeof(int)
    xIntegerQueue = xQueueCreate(10, sizeof(int));
    
    if (xIntegerQueue == NULL) {
        xil_printf("Error: Could not create queue!\r\n");
        return -1;
    }
    
    // Create tasks
    xTaskCreate(vProducerTask, "Producer", 1024, NULL, 2, NULL);
    xTaskCreate(vConsumerTask, "Consumer", 1024, NULL, 1, NULL);
    
    vTaskStartScheduler();
    for (;;);
}
```

![Basic Queue Demo]({{ site.baseurl }}/docs/imgs/labs/lab3-img-002.png)

---

## Part 2: Structured Data Queue

### Sensor Reading Structure

```c
typedef struct {
    uint8_t  sensorID;
    uint16_t value;
    uint32_t timestamp;
} SensorReading_t;

QueueHandle_t xSensorQueue;

void vSensorTask(void *pvParameters) {
    SensorReading_t reading;
    uint8_t sensorID = (uint8_t)(uintptr_t)pvParameters;
    
    for (;;) {
        // Simulate sensor reading
        reading.sensorID = sensorID;
        reading.value = rand() % 4096;  // Simulated 12-bit ADC
        reading.timestamp = xTaskGetTickCount();
        
        xil_printf("Sensor %d: Value=%d, Time=%lu\r\n", 
                   reading.sensorID, reading.value, reading.timestamp);
        
        xQueueSend(xSensorQueue, &reading, pdMS_TO_TICKS(100));
        
        vTaskDelay(pdMS_TO_TICKS(200 + (sensorID * 100)));  // Different rates
    }
}

void vProcessorTask(void *pvParameters) {
    SensorReading_t reading;
    
    for (;;) {
        if (xQueueReceive(xSensorQueue, &reading, portMAX_DELAY) == pdPASS) {
            xil_printf("Processor: Sensor %d = %d @ %lu\r\n",
                       reading.sensorID, reading.value, reading.timestamp);
            
            // Process the data
            if (reading.value > 3000) {
                xil_printf("  WARNING: High reading from sensor %d!\r\n", 
                           reading.sensorID);
            }
        }
    }
}

int main(void) {
    // Create queue for sensor readings
    xSensorQueue = xQueueCreate(20, sizeof(SensorReading_t));
    
    // Create 3 sensor tasks
    xTaskCreate(vSensorTask, "Sensor1", 1024, (void *)1, 2, NULL);
    xTaskCreate(vSensorTask, "Sensor2", 1024, (void *)2, 2, NULL);
    xTaskCreate(vSensorTask, "Sensor3", 1024, (void *)3, 2, NULL);
    
    // Create processor task
    xTaskCreate(vProcessorTask, "Processor", 1024, NULL, 3, NULL);
    
    vTaskStartScheduler();
    for (;;);
}
```

![Structured Queue Demo]({{ site.baseurl }}/docs/imgs/labs/lab3-img-004.png)

---

## Part 3: Multiple Producers

### Button Events with Priority

```c
typedef struct {
    uint8_t buttonID;
    uint8_t eventType;  // 0=press, 1=release, 2=hold
    uint32_t duration;
} ButtonEvent_t;

QueueHandle_t xButtonQueue;

// Button ISR - sends events from interrupt
void ButtonISR_Handler(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    ButtonEvent_t event;
    
    // Determine which button
    u32 buttons = XGpio_DiscreteRead(&GpioButtons, 1);
    
    event.buttonID = (buttons & 0x01) ? 0 : 
                     (buttons & 0x02) ? 1 :
                     (buttons & 0x04) ? 2 : 3;
    event.eventType = 0;  // Press
    event.duration = 0;
    
    // Send to queue from ISR
    xQueueSendFromISR(xButtonQueue, &event, &xHigherPriorityTaskWoken);
    
    // Clear interrupt
    XGpio_InterruptClear(&GpioButtons, XGPIO_IR_CH1_MASK);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void vEventHandlerTask(void *pvParameters) {
    ButtonEvent_t event;
    
    for (;;) {
        if (xQueueReceive(xButtonQueue, &event, portMAX_DELAY) == pdPASS) {
            xil_printf("Button %d: event=%d, duration=%lu\r\n",
                       event.buttonID, event.eventType, event.duration);
            
            // Handle different buttons
            switch (event.buttonID) {
                case 0: xil_printf("  -> Start action\r\n"); break;
                case 1: xil_printf("  -> Stop action\r\n"); break;
                case 2: xil_printf("  -> Mode change\r\n"); break;
                case 3: xil_printf("  -> Reset\r\n"); break;
            }
        }
    }
}
```

![Multiple Producers]({{ site.baseurl }}/docs/imgs/labs/lab3-img-006.png)

---

## Part 4: Queue Sets (Advanced)

### Monitoring Multiple Queues

```c
#include "queue.h"

QueueHandle_t xQueue1, xQueue2;
QueueSetHandle_t xQueueSet;

void vMultiQueueTask(void *pvParameters) {
    QueueSetMemberHandle_t xActivatedMember;
    int receivedValue;
    
    for (;;) {
        // Block on queue set - wakes when ANY queue has data
        xActivatedMember = xQueueSelectFromSet(xQueueSet, portMAX_DELAY);
        
        if (xActivatedMember == xQueue1) {
            xQueueReceive(xQueue1, &receivedValue, 0);
            xil_printf("Queue 1: %d\r\n", receivedValue);
        } else if (xActivatedMember == xQueue2) {
            xQueueReceive(xQueue2, &receivedValue, 0);
            xil_printf("Queue 2: %d\r\n", receivedValue);
        }
    }
}
```

---

## Part 5: Performance Analysis

### Queue Statistics

```c
void vMonitorTask(void *pvParameters) {
    for (;;) {
        UBaseType_t queueLength = uxQueueMessagesWaiting(xSensorQueue);
        UBaseType_t spacesAvailable = uxQueueSpacesAvailable(xSensorQueue);
        
        xil_printf("Queue Status: %d items, %d spaces free\r\n",
                   queueLength, spacesAvailable);
        
        if (queueLength > 15) {
            xil_printf("WARNING: Queue nearly full!\r\n");
        }
        
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## Deliverables

1. **Source Code**:
   - Basic integer queue implementation
   - Structured sensor data queue
   - ISR-to-task queue communication

2. **Screenshots**:
   - Serial output for each part
   - Queue statistics output

3. **Lab Report**:
   - Analysis of producer/consumer timing
   - Queue size selection justification
   - Observations about blocking behavior

---

## Questions to Answer

1. What happens when the producer is faster than the consumer?
2. How does queue size affect system behavior?
3. Why use `xQueueSendFromISR()` instead of `xQueueSend()` in an ISR?
4. What is the difference between `xQueueReceive()` and `xQueuePeek()`?

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Queue creation fails | Increase FreeRTOS heap size |
| Data corruption | Check structure size matches queue item size |
| Timeout on send | Queue full - increase size or speed up consumer |
| Crash in ISR | Use `FromISR` variants, check stack size |

---

## Reference Materials

- [Lab 3 PDF Writeup](../pdfs/LAB%203%20FreeRTOS%20Data%20Queue%20on%20Zybo_2024FALL.pdf)
- [Module 7: Queues and Inter-Task Communication]({{ site.baseurl }}/docs/modules/module07.html)
- [FreeRTOS Queue API Reference](https://www.freertos.org/a00018.html)
