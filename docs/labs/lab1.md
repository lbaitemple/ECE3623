---
title: "Lab 1: Task Management in FreeRTOS"
layout: default
parent: Laboratory Exercises
nav_order: 1
---

# Lab 1: Task Management in FreeRTOS
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will implement and explore FreeRTOS task management on the Zybo Z7 board. You will create multiple tasks with different priorities, observe scheduling behavior, and understand how the RTOS manages concurrent execution.

**Related Modules**: [Module 2: RTOS Fundamentals]({{ site.baseurl }}/docs/modules/module02.html), [Module 3: FreeRTOS Task Management]({{ site.baseurl }}/docs/modules/module03.html)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Create FreeRTOS tasks using `xTaskCreate()`
2. Assign and manage task priorities
3. Observe preemptive scheduling behavior
4. Use `vTaskDelay()` for periodic task execution
5. Monitor task execution through serial output

---

## Prerequisites

- Completion of Module 2 and Module 3 lectures
- Vivado and Vitis installed and configured
- Zybo Z7 board connected and tested

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- Micro-USB cable for programming and UART
- (Optional) Oscilloscope for timing verification

---

## Background

### FreeRTOS Task States

Tasks in FreeRTOS exist in one of four states:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TASK STATE DIAGRAM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌─────────────┐                            │
│           Resume     │  SUSPENDED  │     Suspend                │
│          ┌──────────►│             │◄──────────┐                │
│          │           └─────────────┘           │                │
│          │                 │                   │                │
│          │          vTaskResume()              │                │
│          │                 │                   │                │
│          │                 ▼                   │                │
│          │           ┌─────────────┐           │                │
│          │   ┌──────►│    READY    │◄──────────┼────┐           │
│          │   │       └─────────────┘           │    │           │
│          │   │             │                   │    │ Preempted │
│          │   │     Scheduler runs              │    │           │
│          │   │             │                   │    │           │
│          │   │             ▼                   │    │           │
│          │   │       ┌─────────────┐           │    │           │
│          │   │       │   RUNNING   │───────────┴────┘           │
│          │   │       └─────────────┘                            │
│          │   │             │                                     │
│          │   │        Blocking call                             │
│          │   │        (delay, queue)                            │
│          │   │             │                                     │
│          │   │             ▼                                     │
│          │   │       ┌─────────────┐                            │
│          │   └───────│   BLOCKED   │                            │
│          │  Timeout  └─────────────┘                            │
│          └─────────────────┘                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![FreeRTOS Task States]({{ site.baseurl }}/docs/imgs/labs/lab1-img-000.png)

### Task Priorities

- Priority 0 is the **lowest** (idle task level)
- Higher numbers = higher priority
- Highest priority ready task always runs
- Equal priority tasks share time (round-robin)

---

## Part 1: Project Setup

### Step 1: Create Vivado Project

1. Launch Vivado and create a new project
2. Select Zybo Z7 board as target
3. Create a block design with ZYNQ7 Processing System
4. Configure PS with UART0 enabled
5. Generate bitstream

![Vivado Block Design]({{ site.baseurl }}/docs/imgs/labs/lab1-img-002.png)

### Step 2: Create Vitis Application

1. Export hardware (.xsa file)
2. Launch Vitis and create new platform project
3. Create application project with **FreeRTOS** template
4. Verify FreeRTOS BSP is configured

---

## Part 2: Basic Task Creation

### Task 1: Create Two Alternating Tasks

Create two tasks that print messages to the UART:

```c
#include "FreeRTOS.h"
#include "task.h"
#include "xil_printf.h"

// Task 1: Print "Task A" every 500ms
void vTaskA(void *pvParameters) {
    for (;;) {
        xil_printf("Task A running\r\n");
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

// Task 2: Print "Task B" every 1000ms
void vTaskB(void *pvParameters) {
    for (;;) {
        xil_printf("Task B running\r\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main(void) {
    xil_printf("FreeRTOS Task Demo Starting...\r\n");
    
    // Create Task A - Priority 1
    xTaskCreate(vTaskA, "TaskA", 1024, NULL, 1, NULL);
    
    // Create Task B - Priority 1
    xTaskCreate(vTaskB, "TaskB", 1024, NULL, 1, NULL);
    
    // Start scheduler
    vTaskStartScheduler();
    
    // Should never reach here
    for (;;);
    return 0;
}
```

### Expected Output

```
FreeRTOS Task Demo Starting...
Task A running
Task B running
Task A running
Task A running
Task B running
Task A running
Task A running
Task B running
...
```

---

## Part 3: Priority Experiments

### Task 2: Observe Priority Effects

Modify the tasks to have different priorities:

```c
// Create Task A - Priority 2 (HIGHER)
xTaskCreate(vTaskA, "TaskA", 1024, NULL, 2, NULL);

// Create Task B - Priority 1 (LOWER)
xTaskCreate(vTaskB, "TaskB", 1024, NULL, 1, NULL);
```

### Questions to Answer

1. What happens when Task A has higher priority than Task B?
2. Does Task B ever get to run? Why or why not?
3. What role does `vTaskDelay()` play in allowing lower-priority tasks to run?

![Priority Scheduling]({{ site.baseurl }}/docs/imgs/labs/lab1-img-004.png)

---

## Part 4: Task Parameters

### Task 3: Passing Parameters to Tasks

Create a generic task that uses parameters:

```c
void vGenericTask(void *pvParameters) {
    char *taskName = (char *)pvParameters;
    int count = 0;
    
    for (;;) {
        xil_printf("%s: count = %d\r\n", taskName, count++);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main(void) {
    xil_printf("Parameter Demo Starting...\r\n");
    
    // Create multiple tasks with same function, different names
    xTaskCreate(vGenericTask, "Sensor1", 1024, "Sensor 1", 1, NULL);
    xTaskCreate(vGenericTask, "Sensor2", 1024, "Sensor 2", 1, NULL);
    xTaskCreate(vGenericTask, "Sensor3", 1024, "Sensor 3", 2, NULL);
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Part 5: Task Control

### Task 4: Suspend and Resume Tasks

Implement task suspension and resumption:

```c
TaskHandle_t xTaskBHandle = NULL;

void vTaskA(void *pvParameters) {
    int iteration = 0;
    
    for (;;) {
        xil_printf("Task A: iteration %d\r\n", iteration++);
        
        if (iteration == 5) {
            xil_printf("Task A: Suspending Task B\r\n");
            vTaskSuspend(xTaskBHandle);
        }
        
        if (iteration == 10) {
            xil_printf("Task A: Resuming Task B\r\n");
            vTaskResume(xTaskBHandle);
        }
        
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void vTaskB(void *pvParameters) {
    for (;;) {
        xil_printf("Task B: I'm running!\r\n");
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main(void) {
    xTaskCreate(vTaskA, "TaskA", 1024, NULL, 2, NULL);
    xTaskCreate(vTaskB, "TaskB", 1024, NULL, 1, &xTaskBHandle);
    
    vTaskStartScheduler();
    for (;;);
}
```

---

## Deliverables

1. **Source Code**: Complete C source files for all tasks
2. **Screenshots**:
   - Vivado block design
   - Serial terminal output for each part
3. **Lab Report** including:
   - Answers to all questions
   - Observations about scheduling behavior
   - Analysis of priority effects

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No UART output | Check COM port and baud rate (115200) |
| Tasks don't run | Verify `vTaskStartScheduler()` is called |
| Crash/hang | Check stack sizes (increase if needed) |
| Unexpected scheduling | Verify priority assignments |

---

## Reference Materials

- [Lab 1 PDF Writeup](../pdfs/LAB%201%20Task%20Management%20in%20FreeRTOS%20_2023Fall.pdf)
- [FreeRTOS Implementation Guide](../pdfs/FreeRTOS%20Implementation%20in%20Xilinx%20Vivado%20and%20SDK.pdf)
- [Module 3: FreeRTOS Task Management]({{ site.baseurl }}/docs/modules/module03.html)
