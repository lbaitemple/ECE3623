---
title: "Module 7: Queues and Inter-Task Communication"
layout: default
parent: Course Modules
nav_order: 7
---

# Module 7: Queues and Inter-Task Communication
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the role of queues in FreeRTOS for inter-task communication
2. Create and configure queues using `xQueueCreate()`
3. Send and receive data using `xQueueSend()` and `xQueueReceive()` with appropriate timeouts
4. Use ISR-safe queue operations (`xQueueSendFromISR`)
5. Implement producer-consumer patterns with queues

---

## Overview

Tasks in FreeRTOS often need to exchange data. While shared global variables seem simple, they lead to race conditions (as we saw in Module 4). **Queues** provide a thread-safe mechanism for passing data between tasks—and even from ISRs to tasks.

---

## Why Use Queues?

### The Problem with Shared Variables

```c
// DANGEROUS: Race condition!
volatile int shared_data;
volatile int data_ready = 0;

void ProducerTask(void *param) {
    for (;;) {
        shared_data = ReadSensor();  // Write data
        data_ready = 1;               // Signal ready
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void ConsumerTask(void *param) {
    for (;;) {
        if (data_ready) {
            ProcessData(shared_data);  // May read stale or corrupted data!
            data_ready = 0;
        }
    }
}
```

**Problems**:
- Consumer might read while producer is writing
- Consumer busy-waits (wastes CPU)
- Single item only—what if producer is faster?

### The Queue Solution

```c
QueueHandle_t xDataQueue;

void ProducerTask(void *param) {
    int sensor_value;
    for (;;) {
        sensor_value = ReadSensor();
        xQueueSend(xDataQueue, &sensor_value, portMAX_DELAY);  // Thread-safe!
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void ConsumerTask(void *param) {
    int value;
    for (;;) {
        if (xQueueReceive(xDataQueue, &value, portMAX_DELAY) == pdTRUE) {
            ProcessData(value);  // Data is safe, consumer blocked efficiently
        }
    }
}
```

---

## Queue Fundamentals

### What is a Queue?

A FreeRTOS queue is a **thread-safe FIFO buffer** that:

```
┌─────────────────────────────────────────────────────────────────┐
│                        FreeRTOS Queue                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐     │
│  │ Item │ Item │ Item │      │      │      │      │      │     │
│  │  1   │  2   │  3   │      │      │      │      │      │     │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘     │
│     ↑                    ↑                                       │
│    Head                 Tail                                     │
│  (Receive)             (Send)                                    │
│                                                                  │
│  • Fixed item size (set at creation)                            │
│  • Fixed maximum length (set at creation)                        │
│  • Items copied by VALUE (not reference)                         │
│  • Blocking operations with timeout                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Thread-safe** | No additional locking needed |
| **Copy by value** | Items are copied into/out of queue |
| **Blocking** | Tasks can block waiting for space or data |
| **Timeout** | Operations can timeout |
| **Priority-aware** | Highest priority blocked task unblocks first |
| **ISR-safe** | Special `FromISR` versions available |

---

## Creating Queues

### xQueueCreate()

```c
QueueHandle_t xQueueCreate(
    UBaseType_t uxQueueLength,   // Maximum number of items
    UBaseType_t uxItemSize       // Size of each item in bytes
);
```

### Example: Creating Different Queue Types

```c
#include "FreeRTOS.h"
#include "queue.h"

// Queue of integers (10 items, 4 bytes each)
QueueHandle_t xIntQueue = xQueueCreate(10, sizeof(int));

// Queue of sensor readings
typedef struct {
    uint16_t sensor_id;
    float value;
    TickType_t timestamp;
} SensorReading_t;

QueueHandle_t xSensorQueue = xQueueCreate(20, sizeof(SensorReading_t));

// Queue of pointers (for large data)
QueueHandle_t xPointerQueue = xQueueCreate(5, sizeof(char *));

// Always check for success!
if (xIntQueue == NULL) {
    xil_printf("Error: Could not create queue\r\n");
}
```

### Memory Considerations

Queue memory = (uxQueueLength × uxItemSize) + queue overhead (~76 bytes)

| Queue | Length | Item Size | Total RAM |
|-------|--------|-----------|-----------|
| Int queue | 10 | 4 bytes | ~116 bytes |
| Sensor queue | 20 | 12 bytes | ~316 bytes |
| Large buffer queue | 5 | 1024 bytes | ~5196 bytes |

**Tip**: For large data, queue pointers to statically allocated buffers.

---

## Sending to Queues

### xQueueSend() / xQueueSendToBack()

```c
BaseType_t xQueueSend(
    QueueHandle_t xQueue,        // Queue handle
    const void *pvItemToQueue,   // Pointer to item to copy
    TickType_t xTicksToWait      // Timeout in ticks
);
```

### Return Values

| Return Value | Meaning |
|--------------|---------|
| `pdTRUE` / `pdPASS` | Item successfully sent |
| `errQUEUE_FULL` | Queue full, timeout expired |

### Timeout Options

```c
// Block forever until space available
xQueueSend(xQueue, &data, portMAX_DELAY);

// Block up to 100ms
xQueueSend(xQueue, &data, pdMS_TO_TICKS(100));

// Don't block at all (return immediately)
if (xQueueSend(xQueue, &data, 0) != pdTRUE) {
    // Queue was full, handle error
    xil_printf("Warning: Queue full, data discarded\r\n");
}
```

### Send to Front (Priority Data)

```c
// Insert at front of queue (bypasses FIFO for urgent data)
xQueueSendToFront(xQueue, &urgent_data, portMAX_DELAY);
```

### Overwriting (Queue of Length 1)

For latest-value scenarios (e.g., current sensor reading):

```c
QueueHandle_t xLatestValue = xQueueCreate(1, sizeof(int));

// Always succeeds - overwrites if full
xQueueOverwrite(xLatestValue, &new_reading);
```

---

## Receiving from Queues

### xQueueReceive()

```c
BaseType_t xQueueReceive(
    QueueHandle_t xQueue,        // Queue handle
    void *pvBuffer,              // Buffer to receive item
    TickType_t xTicksToWait      // Timeout in ticks
);
```

### Example Usage

```c
int received_value;

// Block until data available (efficient waiting)
if (xQueueReceive(xQueue, &received_value, portMAX_DELAY) == pdTRUE) {
    xil_printf("Received: %d\r\n", received_value);
}

// Block with timeout
if (xQueueReceive(xQueue, &received_value, pdMS_TO_TICKS(500)) == pdTRUE) {
    ProcessData(received_value);
} else {
    xil_printf("Timeout: No data received in 500ms\r\n");
}
```

### Peeking (Read Without Removing)

```c
int peeked_value;

// Look at front item without removing it
if (xQueuePeek(xQueue, &peeked_value, 0) == pdTRUE) {
    xil_printf("Front item is: %d\r\n", peeked_value);
}
```

### Query Queue Status

```c
// Number of items currently in queue
UBaseType_t count = uxQueueMessagesWaiting(xQueue);

// Available space in queue
UBaseType_t space = uxQueueSpacesAvailable(xQueue);

xil_printf("Queue: %d items, %d spaces\r\n", count, space);
```

---

## Queue Operations from ISRs

ISRs must use special `FromISR` versions that never block:

### xQueueSendFromISR()

```c
void ADC_ISR(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // Read ADC value
    uint16_t adc_value = ReadADCRegister();
    
    // Send to queue (non-blocking)
    xQueueSendFromISR(xADCQueue, &adc_value, &xHigherPriorityTaskWoken);
    
    // Yield if we woke a higher priority task
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### xQueueReceiveFromISR()

```c
void TX_ISR(void *CallbackRef) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t byte_to_send;
    
    // Get next byte from queue
    if (xQueueReceiveFromISR(xTxQueue, &byte_to_send, 
                              &xHigherPriorityTaskWoken) == pdTRUE) {
        SendByte(byte_to_send);
    } else {
        // Queue empty, disable TX interrupt
        DisableTxInterrupt();
    }
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### Why the Extra Parameter?

`xHigherPriorityTaskWoken` indicates if a higher-priority task was unblocked. If `pdTRUE`, we should yield so that task can run immediately:

```c
// This ensures responsive handling:
// ISR → Queue Send → Unblocks High Priority Task → Immediate Context Switch
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

---

## Producer-Consumer Pattern

The classic multi-task communication pattern:

### Single Producer, Single Consumer

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "xil_printf.h"

#define QUEUE_LENGTH    10
#define QUEUE_ITEM_SIZE sizeof(int)

QueueHandle_t xDataQueue;

// Producer: Generates data
void vProducerTask(void *param) {
    int counter = 0;
    
    for (;;) {
        // Produce data
        int data = counter++;
        
        // Send to queue
        if (xQueueSend(xDataQueue, &data, pdMS_TO_TICKS(100)) == pdTRUE) {
            xil_printf("Produced: %d\r\n", data);
        } else {
            xil_printf("Producer: Queue full!\r\n");
        }
        
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

// Consumer: Processes data
void vConsumerTask(void *param) {
    int received;
    
    for (;;) {
        // Wait for data
        if (xQueueReceive(xDataQueue, &received, portMAX_DELAY) == pdTRUE) {
            xil_printf("Consumed: %d\r\n", received);
            
            // Simulate processing time
            vTaskDelay(pdMS_TO_TICKS(500));
        }
    }
}

int main(void) {
    // Create queue
    xDataQueue = xQueueCreate(QUEUE_LENGTH, QUEUE_ITEM_SIZE);
    configASSERT(xDataQueue != NULL);
    
    // Create tasks
    xTaskCreate(vProducerTask, "Producer", 256, NULL, 2, NULL);
    xTaskCreate(vConsumerTask, "Consumer", 256, NULL, 1, NULL);
    
    vTaskStartScheduler();
    for (;;);
}
```

### Multiple Producers, Single Consumer

```c
// Multiple sensor tasks sending to one processing task

typedef struct {
    uint8_t sensor_id;
    uint16_t value;
} SensorData_t;

QueueHandle_t xSensorQueue;

void vSensor1Task(void *param) {
    SensorData_t data = { .sensor_id = 1 };
    for (;;) {
        data.value = ReadSensor1();
        xQueueSend(xSensorQueue, &data, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void vSensor2Task(void *param) {
    SensorData_t data = { .sensor_id = 2 };
    for (;;) {
        data.value = ReadSensor2();
        xQueueSend(xSensorQueue, &data, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

void vProcessorTask(void *param) {
    SensorData_t received;
    for (;;) {
        if (xQueueReceive(xSensorQueue, &received, portMAX_DELAY) == pdTRUE) {
            xil_printf("Sensor %d: %d\r\n", received.sensor_id, received.value);
            ProcessReading(&received);
        }
    }
}
```

---

## Queue Sets (Advanced)

Wait on multiple queues simultaneously:

```c
#include "queue.h"

QueueSetHandle_t xQueueSet;
QueueHandle_t xQueue1, xQueue2;

void setup(void) {
    // Create queues
    xQueue1 = xQueueCreate(10, sizeof(int));
    xQueue2 = xQueueCreate(10, sizeof(int));
    
    // Create queue set (total capacity must cover all members)
    xQueueSet = xQueueCreateSet(20);
    
    // Add queues to set
    xQueueAddToSet(xQueue1, xQueueSet);
    xQueueAddToSet(xQueue2, xQueueSet);
}

void vMultiQueueTask(void *param) {
    QueueSetMemberHandle_t xActiveMember;
    int value;
    
    for (;;) {
        // Block until ANY queue has data
        xActiveMember = xQueueSelectFromSet(xQueueSet, portMAX_DELAY);
        
        if (xActiveMember == xQueue1) {
            xQueueReceive(xQueue1, &value, 0);
            xil_printf("From Queue1: %d\r\n", value);
        }
        else if (xActiveMember == xQueue2) {
            xQueueReceive(xQueue2, &value, 0);
            xil_printf("From Queue2: %d\r\n", value);
        }
    }
}
```

---

## Best Practices

### Do's and Don'ts

| ✅ Do | ❌ Don't |
|------|---------|
| Choose appropriate queue length | Create huge queues "just in case" |
| Use timeouts to detect problems | Always use `portMAX_DELAY` |
| Check return values | Ignore failed sends |
| Use `FromISR` in ISRs | Call `xQueueSend` in ISR |
| Size items appropriately | Queue large buffers by value |

### Choosing Queue Length

| Scenario | Suggested Length |
|----------|-----------------|
| ISR to task (bursty) | 10-20 items |
| Steady producer/consumer | 5-10 items |
| Latest value only | 1 item (use `xQueueOverwrite`) |
| Command queue | 5-10 commands |

---

## Lecture Materials

- [FreeRTOS Queue Management]({{ site.baseurl }}/ece3623/FreeRTOS.pdf)
- [Week 7 Slides]({{ site.baseurl }}/ece3623/Week%207.pdf)
- [Week 7 Recap]({{ site.baseurl }}/ece3623/Week%207-R.pdf)

---

## Reading Assignments

1. *Mastering the FreeRTOS Real Time Kernel*, Chapter 4: Queue Management
2. FreeRTOS Documentation: Queue API Reference
3. FreeRTOS Documentation: Queue Sets

---

## Practice Questions

1. What are the advantages of using queues over shared global variables?
2. What happens if a task calls `xQueueReceive()` on an empty queue with `portMAX_DELAY`?
3. Why must ISRs use `xQueueSendFromISR()` instead of `xQueueSend()`?
4. A queue is created with length 5 and item size 4. How much RAM does it use (approximately)?
5. What is the difference between `xQueueSend()` and `xQueueSendToFront()`?
6. When would you use `xQueueOverwrite()` instead of `xQueueSend()`?
7. Explain the purpose of the `xHigherPriorityTaskWoken` parameter in ISR-safe queue functions.

---

## Summary

Queues are the primary mechanism for thread-safe data transfer between FreeRTOS tasks. They provide blocking operations with timeout, priority-aware task unblocking, and ISR-safe variants. The producer-consumer pattern using queues is fundamental to many embedded designs. Understanding queue sizing, timeout selection, and proper ISR usage is essential for building reliable multi-task applications.

---

## Next Module

[Module 8: Software Timers and Watchdog →](../module08/)

---

## Lab Connection

### Lab 2: Vivado AXI Interrupt

In this lab, you will:
1. Create a block design with AXI GPIO
2. Configure fabric interrupt connection
3. Write an ISR for button press
4. Implement LED toggle on interrupt

**Lab Materials**: [Lab 2 - AXI Interrupt]({{ site.baseurl }}/ece3623/LAB%202%20Vivado%20AXI%20Interrupt%20.pdf)

---

## Lecture Materials

- [Week 5 Slides]({{ site.baseurl }}/ece3623/Week%205-T.pdf)
- [Week 5 Additional]({{ site.baseurl }}/ece3623/Week%205-T%20(1).pdf)

---

## Reading Assignments

1. Zynq-7000 TRM, Chapter 7: Interrupts
2. ARM GIC Architecture Specification (overview)
3. FreeRTOS documentation: Interrupt Management

---

## Practice Questions

1. What is the difference between IRQ and FIQ?

2. An interrupt has ID 65. Is this an SGI, PPI, or SPI?

3. Why should you avoid calling `printf()` from an ISR?

4. Write pseudocode for an ISR that signals a FreeRTOS task using a semaphore.

5. A system has interrupts at priorities 0xA0, 0x80, and 0xC0. Which has highest priority? (Lower number = higher priority in GIC)

---

## Summary

Interrupts enable responsive event handling without wasting CPU cycles on polling. We covered ARM interrupt architecture, GIC configuration, ISR design, and FreeRTOS integration. Proper interrupt handling is crucial for real-time system performance. The next module covers inter-task communication mechanisms.

---

## Next Module

[Module 8: Inter-Task Communication →](../module08/)
