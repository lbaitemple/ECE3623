---
title: "Module 14: HLS Directives and Optimization"
layout: default
parent: "Progression 5: Hardware Acceleration"
grand_parent: "Course Modules"
nav_order: 2
---

# Module 14: HLS Directives and Optimization
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Apply HLS directives including PIPELINE, UNROLL, and DATAFLOW
2. Reduce latency (time for one iteration) using optimization techniques
3. Improve throughput (rate of data processed) through parallelism
4. Enable concurrent block operations with DATAFLOW
5. Analyze performance tradeoffs between latency, throughput, and resources

---

## Overview

To achieve high performance in HLS designs, optimization directives (pragmas) guide the synthesis tool to generate more efficient hardware. This module teaches how to use PIPELINE, UNROLL, and DATAFLOW directives to reduce latency and maximize throughput.

---

## Performance Metrics

### Latency vs Throughput

| Metric | Definition | Goal |
|--------|------------|------|
| **Latency** | Clock cycles from first input to first output | Minimize for fast response |
| **Initiation Interval (II)** | Clock cycles between consecutive inputs | Minimize for high throughput |
| **Throughput** | Data samples processed per unit time | Maximize for high bandwidth |

### Relationship

$$\text{Throughput} = \frac{f_{clk}}{II}$$

Where:
- $f_{clk}$ = clock frequency (Hz)
- $II$ = initiation interval (cycles)

**Example**: 100 MHz clock, II=1 → Throughput = 100 MSPS

---

## Baseline Example: Unoptimized FIR

```c
#define N 16

void fir_baseline(int *y, int x) {
    static int shift_reg[N];
    static const int c[N] = {1,2,3,4,5,6,7,8,8,7,6,5,4,3,2,1};
    int acc = 0;
    
    // Shift register
    for (int i = N-1; i > 0; i--) {
        shift_reg[i] = shift_reg[i-1];
    }
    shift_reg[0] = x;
    
    // MAC operations
    for (int i = 0; i < N; i++) {
        acc += shift_reg[i] * c[i];
    }
    
    *y = acc;
}
```

### Baseline Performance

```
Without optimization:
- Latency: ~48 cycles
- Interval (II): ~48 cycles
- Throughput: 1 sample per 48 cycles
```

---

## PIPELINE Directive

### Purpose

Execute loop iterations in an **overlapping** manner, allowing new iterations to begin before previous ones complete.

### Syntax

```c
#pragma HLS PIPELINE II=<target>
```

### How Pipelining Works

**Without Pipeline** (Sequential):
```
Sample 1: │████████████████│
Sample 2:                   │████████████████│
Sample 3:                                     │████████████████│
          0    5   10   15   20   25   30   35   40   45   50
                         Clock Cycles
```

**With Pipeline (II=1)**:
```
Sample 1: │████████████████│
Sample 2:  │████████████████│
Sample 3:   │████████████████│
           0    5   10   15   20
                Clock Cycles

Latency still = 16 cycles, but II = 1 cycle!
```

### Pipelined FIR Filter

```c
void fir_pipelined(int *y, int x) {
    static int shift_reg[N];
    static const int c[N] = {1,2,3,4,5,6,7,8,8,7,6,5,4,3,2,1};
    int acc = 0;
    
    #pragma HLS PIPELINE II=1
    
    Shift_Loop:
    for (int i = N-1; i > 0; i--) {
        #pragma HLS UNROLL
        shift_reg[i] = shift_reg[i-1];
    }
    shift_reg[0] = x;
    
    MAC_Loop:
    for (int i = 0; i < N; i++) {
        #pragma HLS UNROLL
        acc += shift_reg[i] * c[i];
    }
    
    *y = acc;
}
```

### Performance After Pipelining

```
With PIPELINE II=1:
- Latency: 3 cycles
- Interval (II): 1 cycle
- Throughput: 1 sample per cycle (100 MSPS @ 100MHz)
```

---

## UNROLL Directive

### Purpose

Replicate loop body multiple times to enable parallel execution.

### Syntax

```c
#pragma HLS UNROLL              // Full unroll
#pragma HLS UNROLL factor=N     // Partial unroll by factor N
```

### Unrolling Visualization

**Rolled Loop** (1 multiplier, sequential):
```c
for (int i = 0; i < 4; i++) {
    acc += a[i] * b[i];  // 4 iterations, 1 multiplier
}
```
```
Cycle: │ 1 │ 2 │ 3 │ 4 │
       │MUL│MUL│MUL│MUL│  ← Same multiplier reused
```

**Fully Unrolled** (4 multipliers, parallel):
```c
#pragma HLS UNROLL
for (int i = 0; i < 4; i++) {
    acc += a[i] * b[i];  // 4 multipliers in parallel
}
```
```
Cycle: │ 1 │
       │MUL│  ← Multiplier 0
       │MUL│  ← Multiplier 1
       │MUL│  ← Multiplier 2
       │MUL│  ← Multiplier 3
       (All execute simultaneously)
```

### Partial Unrolling

```c
// Original: 16 iterations
for (int i = 0; i < 16; i++) {
    acc += data[i];
}

// Partial unroll factor=4: 4 iterations, each does 4 ops
#pragma HLS UNROLL factor=4
for (int i = 0; i < 16; i++) {
    acc += data[i];
}
```

| Unroll Factor | Iterations | Parallel Ops | Latency |
|---------------|------------|--------------|---------|
| 1 (none) | 16 | 1 | 16 cycles |
| 4 | 4 | 4 | 4 cycles |
| 16 (full) | 1 | 16 | 1 cycle |

---

## ARRAY_PARTITION Directive

### Purpose

Split arrays to enable parallel access (required for unrolled loops).

### The Problem

```c
#pragma HLS UNROLL
for (int i = 0; i < 4; i++) {
    acc += a[i] * b[i];  // Want 4 parallel reads
}
// BUT: BRAM only has 2 ports → bottleneck!
```

### Solution: Array Partitioning

```c
#pragma HLS ARRAY_PARTITION variable=a complete
#pragma HLS ARRAY_PARTITION variable=b complete
#pragma HLS UNROLL
for (int i = 0; i < 4; i++) {
    acc += a[i] * b[i];  // Now 4 parallel reads possible
}
```

### Partition Types

```
COMPLETE: Split into individual registers
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  Original array
└───┴───┴───┴───┘
       ↓
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ 0 │ │ 1 │ │ 2 │ │ 3 │  Individual registers
└───┘ └───┘ └───┘ └───┘

CYCLIC factor=2: Interleaved distribution
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  Original array
└───┴───┴───┴───┘
       ↓
┌───┬───┐  ┌───┬───┐
│ 0 │ 2 │  │ 1 │ 3 │  Two BRAMs
└───┴───┘  └───┴───┘

BLOCK factor=2: Contiguous blocks
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  Original array
└───┴───┴───┴───┘
       ↓
┌───┬───┐  ┌───┬───┐
│ 0 │ 1 │  │ 2 │ 3 │  Two BRAMs
└───┴───┘  └───┴───┘
```

---

## DATAFLOW Directive

### Purpose

Enable **task-level pipelining** where multiple functions execute concurrently on different data.

### Syntax

```c
#pragma HLS DATAFLOW
```

### Sequential vs Dataflow

**Sequential Execution**:
```
         │ Process Sample 1 │        │ Process Sample 2 │
─────────┴──────────────────┴────────┴──────────────────┴─────►
         │ A │ B │ C │              │ A │ B │ C │
         
         Total time = 2 × (A + B + C)
```

**Dataflow Execution**:
```
Block A: │███│   │███│   │███│
Block B:    │███│   │███│   │███│
Block C:       │███│   │███│   │███│
         └─────────────────────────────►
                Time
         
         Blocks A, B, C process different samples concurrently!
```

### Dataflow Example

```c
void top_function(int *input, int *output, int n) {
    #pragma HLS DATAFLOW
    
    int temp1[1024];
    int temp2[1024];
    
    // These three functions execute concurrently on different data
    stage1_read(input, temp1, n);
    stage2_process(temp1, temp2, n);
    stage3_write(temp2, output, n);
}

void stage1_read(int *in, int *out, int n) {
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < n; i++) {
        out[i] = in[i];
    }
}

void stage2_process(int *in, int *out, int n) {
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2 + 1;
    }
}

void stage3_write(int *in, int *out, int n) {
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < n; i++) {
        out[i] = in[i];
    }
}
```

### Dataflow Requirements

1. Single-producer, single-consumer between stages
2. No feedback loops
3. Data flows in one direction
4. Use FIFO or PIPO buffers between stages

---

## Optimization Strategy

### Step-by-Step Optimization

```
┌─────────────────────────────────────────────────────────────────┐
│                   OPTIMIZATION WORKFLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Start with working C code (verify correctness)              │
│                     ↓                                            │
│  2. Run C synthesis (baseline performance)                      │
│                     ↓                                            │
│  3. Identify bottleneck (loop? memory? dependency?)             │
│                     ↓                                            │
│  4. Apply appropriate directive:                                │
│     • Loop bottleneck → PIPELINE                                │
│     • Sequential ops → UNROLL                                   │
│     • Memory access → ARRAY_PARTITION                           │
│     • Multi-stage → DATAFLOW                                    │
│                     ↓                                            │
│  5. Re-synthesize and analyze                                   │
│                     ↓                                            │
│  6. Repeat until targets met                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Resource-Performance Tradeoff

| Optimization | Latency Impact | Resource Impact |
|--------------|----------------|-----------------|
| PIPELINE | ↓ II | ↑ Registers |
| UNROLL | ↓ Latency | ↑↑ Logic, multipliers |
| ARRAY_PARTITION | Enables parallelism | ↑ BRAM/FF |
| DATAFLOW | ↓ Total time | ↑ FIFOs |

---

## Complete Optimized Example

### Image Processing Pipeline

```c
#include <hls_stream.h>
#include <ap_int.h>

#define WIDTH  640
#define HEIGHT 480

typedef ap_uint<8> pixel_t;
typedef hls::stream<pixel_t> pixel_stream;

void image_pipeline(pixel_stream &input, pixel_stream &output) {
    #pragma HLS INTERFACE axis port=input
    #pragma HLS INTERFACE axis port=output
    #pragma HLS DATAFLOW
    
    pixel_stream temp1, temp2;
    
    // Stage 1: Read and threshold
    threshold_stage(input, temp1);
    
    // Stage 2: Apply filter
    filter_stage(temp1, temp2);
    
    // Stage 3: Output
    output_stage(temp2, output);
}

void threshold_stage(pixel_stream &in, pixel_stream &out) {
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        pixel_t p = in.read();
        out.write((p > 128) ? 255 : 0);
    }
}

void filter_stage(pixel_stream &in, pixel_stream &out) {
    static pixel_t line_buffer[WIDTH];
    #pragma HLS ARRAY_PARTITION variable=line_buffer cyclic factor=3
    
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        pixel_t p = in.read();
        // Simple 3-tap filter
        pixel_t filtered = (line_buffer[i % WIDTH] + p + line_buffer[(i+1) % WIDTH]) / 3;
        line_buffer[i % WIDTH] = p;
        out.write(filtered);
    }
}

void output_stage(pixel_stream &in, pixel_stream &out) {
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        out.write(in.read());
    }
}
```

---

## Directive Summary

| Directive | Purpose | When to Use |
|-----------|---------|-------------|
| `PIPELINE II=n` | Overlap loop iterations | Inner loops, streaming |
| `UNROLL factor=n` | Parallel loop execution | Small loops, MAC ops |
| `ARRAY_PARTITION` | Enable parallel memory access | With UNROLL |
| `DATAFLOW` | Concurrent function execution | Multi-stage pipelines |
| `INLINE` | Merge function into caller | Small functions |
| `LATENCY min/max` | Constrain timing | Specific requirements |

---

## Lecture Materials

- [Week 14 - HLS Optimization]({{ site.baseurl }}/docs/pdfs/Week%2014%20HLS%20Optimization.pdf)
- [HLS Directives Reference]({{ site.baseurl }}/docs/pdfs/HLS%20Directives.pdf)
- [Performance Optimization Guide]({{ site.baseurl }}/docs/pdfs/HLS%20Performance.pdf)

---

## Reading Assignments

1. Xilinx UG1399: Vitis HLS User Guide, Chapters 4-6 (Optimization)
2. *The Zynq Book*, Chapter 15: HLS Optimization
3. Xilinx HLS optimization tutorials

---

## Practice Questions

1. What is the difference between latency and initiation interval (II)?

2. A function has latency=10 cycles and II=2. How many cycles to process 100 samples?

3. Explain why ARRAY_PARTITION is often required with UNROLL.

4. What are the requirements for using the DATAFLOW directive?

5. A loop has 16 iterations and takes 16 cycles. After adding `#pragma HLS PIPELINE II=1`, what is the new latency and II?

6. When would you use partial unrolling instead of full unrolling?

7. Compare the resource usage between a fully unrolled loop and a pipelined loop.

---

## Summary

HLS optimization directives transform sequential C code into high-performance parallel hardware. PIPELINE enables overlapped execution to achieve II=1 (one output per cycle). UNROLL replicates hardware for parallel computation. ARRAY_PARTITION enables parallel memory access required by unrolled loops. DATAFLOW enables concurrent execution of multiple processing stages. Effective optimization requires understanding the tradeoff between performance (latency, throughput) and resources (DSPs, BRAMs, LUTs).

---

## Course Conclusion

Congratulations on completing ECE 3623 Embedded Systems!

Throughout this course, you have learned:

| Module | Topic |
|--------|-------|
| 1-3 | Zynq architecture, RTOS fundamentals, task management |
| 4-6 | Mutual exclusion, priority inheritance, interrupts |
| 7-9 | Queues, timers, watchdog, SPI/ADC |
| 10-11 | ADC sampling theory, DAC waveform generation |
| 12 | FIR/IIR digital filters, z-transform |
| 13-14 | HLS design flow and optimization |

These skills prepare you for careers in:
- Embedded Systems Engineering
- FPGA/SoC Development
- Real-Time Systems Design
- IoT and Edge Computing
- Automotive and Aerospace Electronics

**Best of luck in your future endeavors!**
┌─────────────────────────────────────────────────────────────────┐
│                          Zynq SoC                                │
├─────────────────────────────────┬───────────────────────────────┤
│         Processing System        │      Programmable Logic       │
│                                  │                               │
│  ┌────────────┐  ┌────────────┐ │  ┌────────────┐  ┌──────────┐│
│  │ FreeRTOS   │  │   UART     │ │  │ AXI GPIO   │  │ HLS FIR  ││
│  │  Tasks     │  │  Driver    │ │  │            │  │  Filter  ││
│  └─────┬──────┘  └─────┬──────┘ │  └─────┬──────┘  └────┬─────┘│
│        │               │         │        │              │       │
│  ┌─────┴───────────────┴─────┐  │  ┌─────┴──────────────┴─────┐│
│  │       AXI Interconnect     │◄─┼─►│     AXI Interconnect     ││
│  └─────────────┬─────────────┘  │  └─────────────┬───────────┘│
│                │                 │                │             │
│  ┌─────────────┴─────────────┐  │  ┌─────────────┴───────────┐│
│  │     PS SPI Controller      │  │  │      AXI SPI (DAC)      ││
│  └─────────────┬─────────────┘  │  └─────────────┬───────────┘│
└────────────────┼─────────────────┴───────────────┼─────────────┘
                 │                                   │
                 ▼                                   ▼
            ┌─────────┐                         ┌─────────┐
            │ PmodAD1 │                         │ PmodDA2 │
            │  (ADC)  │                         │  (DAC)  │
            └─────────┘                         └─────────┘
```

---

## Task Design for FreeRTOS

### Task Architecture

```c
/*
 * System Tasks:
 * - SensorTask: Reads ADC at fixed rate
 * - FilterTask: Applies digital filter
 * - OutputTask: Writes to DAC
 * - CommandTask: Handles UART commands
 * - WatchdogTask: System monitoring
 */

#define PRIORITY_SENSOR     (tskIDLE_PRIORITY + 4)
#define PRIORITY_FILTER     (tskIDLE_PRIORITY + 3)
#define PRIORITY_OUTPUT     (tskIDLE_PRIORITY + 3)
#define PRIORITY_COMMAND    (tskIDLE_PRIORITY + 2)
#define PRIORITY_WATCHDOG   (tskIDLE_PRIORITY + 5)

/* Inter-task communication */
QueueHandle_t xSensorQueue;     /* Raw ADC data */
QueueHandle_t xFilterQueue;     /* Filtered data */
SemaphoreHandle_t xUartMutex;   /* UART protection */
```

### Task Timing Analysis

| Task | Period | WCET | Utilization |
|------|--------|------|-------------|
| Sensor | 1 ms | 50 µs | 5% |
| Filter | 1 ms | 200 µs | 20% |
| Output | 1 ms | 30 µs | 3% |
| Command | 10 ms | 100 µs | 1% |
| Watchdog | 100 ms | 10 µs | 0.01% |
| **Total** | | | **29%** |

With 29% CPU utilization, the system has significant margin.

---

## Complete System Example

### Initialization

```c
int main(void)
{
    /* Hardware initialization */
    Platform_Init();
    UART_Init();
    SPI_Init();
    ADC_Init();
    DAC_Init();
    InterruptSystem_Init();
    Watchdog_Init(1000);
    
    /* Create queues */
    xSensorQueue = xQueueCreate(32, sizeof(u16));
    xFilterQueue = xQueueCreate(32, sizeof(float));
    xUartMutex = xSemaphoreCreateMutex();
    
    /* Create tasks */
    xTaskCreate(vSensorTask, "Sensor", 512, NULL, 
                PRIORITY_SENSOR, NULL);
    xTaskCreate(vFilterTask, "Filter", 1024, NULL, 
                PRIORITY_FILTER, NULL);
    xTaskCreate(vOutputTask, "Output", 512, NULL, 
                PRIORITY_OUTPUT, NULL);
    xTaskCreate(vCommandTask, "Command", 1024, NULL, 
                PRIORITY_COMMAND, NULL);
    xTaskCreate(vWatchdogTask, "Watchdog", 256, NULL, 
                PRIORITY_WATCHDOG, NULL);
    
    /* Start scheduler */
    Watchdog_Start();
    vTaskStartScheduler();
    
    /* Should never reach here */
    for(;;);
    return 0;
}
```

### Sensor Task

```c
void vSensorTask(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(1);
    
    for(;;) {
        /* Read ADC */
        u16 adcValue = ADC_Read();
        
        /* Send to filter task */
        xQueueSend(xSensorQueue, &adcValue, 0);
        
        /* Signal alive for watchdog */
        taskAliveFlags |= SENSOR_ALIVE;
        
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
```

### Filter Task

```c
void vFilterTask(void *pvParameters)
{
    u16 rawValue;
    float filteredValue;
    
    for(;;) {
        /* Wait for data from sensor */
        if (xQueueReceive(xSensorQueue, &rawValue, portMAX_DELAY)) {
            
            /* Convert to voltage */
            float voltage = ADC_ToVoltage(rawValue, 3.3f);
            
            /* Apply filter */
            filteredValue = FIR_Filter(voltage);
            
            /* Send to output task */
            xQueueSend(xFilterQueue, &filteredValue, 0);
            
            taskAliveFlags |= FILTER_ALIVE;
        }
    }
}
```

### Output Task

```c
void vOutputTask(void *pvParameters)
{
    float outputValue;
    
    for(;;) {
        if (xQueueReceive(xFilterQueue, &outputValue, portMAX_DELAY)) {
            
            /* Write to DAC */
            DAC_SetVoltage(outputValue, 3.3f);
            
            taskAliveFlags |= OUTPUT_ALIVE;
        }
    }
}
```

### Command Task

```c
void vCommandTask(void *pvParameters)
{
    char cmdBuffer[64];
    int cmdIndex = 0;
    
    for(;;) {
        if (UART_DataAvailable()) {
            char c = UART_ReceiveByte();
            
            if (c == '\n' || c == '\r') {
                cmdBuffer[cmdIndex] = '\0';
                ProcessCommand(cmdBuffer);
                cmdIndex = 0;
            } else if (cmdIndex < 63) {
                cmdBuffer[cmdIndex++] = c;
            }
        }
        
        taskAliveFlags |= COMMAND_ALIVE;
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

### Watchdog Task

```c
#define ALL_TASKS_ALIVE (SENSOR_ALIVE | FILTER_ALIVE | \
                         OUTPUT_ALIVE | COMMAND_ALIVE)

volatile u8 taskAliveFlags = 0;

void vWatchdogTask(void *pvParameters)
{
    for(;;) {
        if ((taskAliveFlags & ALL_TASKS_ALIVE) == ALL_TASKS_ALIVE) {
            Watchdog_Kick();
            taskAliveFlags = 0;
        }
        /* If not all tasks alive, don't kick - system resets */
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## Debugging Strategies

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Stack Overflow | Crash, corruption | Increase stack, use `uxTaskGetStackHighWaterMark()` |
| Priority Inversion | High-priority blocked | Use mutexes with inheritance |
| Deadlock | System freezes | Check lock ordering |
| Race Condition | Intermittent data corruption | Use queues/mutexes |
| Missed Deadlines | Late responses | Analyze timing, reduce WCET |

### Debugging Tools

```c
/* Stack monitoring */
void PrintTaskStats(void)
{
    TaskHandle_t tasks[] = {xSensorHandle, xFilterHandle, ...};
    
    for (int i = 0; i < NUM_TASKS; i++) {
        UBaseType_t hwm = uxTaskGetStackHighWaterMark(tasks[i]);
        xil_printf("%s: %d words free\r\n", 
                   pcTaskGetName(tasks[i]), hwm);
    }
}

/* Runtime statistics */
void PrintRuntimeStats(void)
{
    char buffer[512];
    vTaskGetRunTimeStats(buffer);
    xil_printf("%s\r\n", buffer);
}
```

---

## Performance Optimization

### Software Optimizations

1. **Reduce blocking time**: Use non-blocking or timeout calls
2. **Optimize algorithms**: Choose O(n) over O(n²)
3. **Use fixed-point**: Faster than floating-point
4. **Minimize memory copies**: Use pointers, DMA

### Hardware Acceleration

Move compute-intensive tasks to PL:
- FIR/IIR filtering
- FFT computation
- Image processing
- Encryption

---

## Project Documentation

### Required Documentation

1. **Requirements Document**: What the system must do
2. **Design Document**: How it's built (block diagrams, task design)
3. **User Manual**: How to use the system
4. **Test Report**: Verification results

### Code Documentation

```c
/**
 * @brief Reads analog value from specified ADC channel
 * 
 * @param channel ADC channel (0 or 1)
 * @return u16 12-bit ADC value (0-4095)
 * 
 * @note Uses SPI to communicate with PmodAD1
 * @warning Must call ADC_Init() before first use
 */
u16 ADC_ReadChannel(u8 channel);
```

---

## Final Project Guidelines

### Project Scope

Design and implement a complete embedded system that:
1. Uses FreeRTOS for task management
2. Interfaces with at least one analog peripheral (ADC or DAC)
3. Implements serial communication (UART or SPI)
4. Includes interrupt handling
5. Demonstrates real-time constraints

### Example Projects

- **Data Logger**: Sample sensors, store data, retrieve via UART
- **Function Generator**: Generate waveforms with frequency/amplitude control
- **Audio Effects Processor**: Real-time filtering with adjustable parameters
- **Motor Controller**: PID control with position feedback
- **Spectrum Analyzer**: FFT display of input signal

### Deliverables

1. Working demonstration
2. Source code (well-documented)
3. Design report
4. Presentation

---

## Lecture Materials

- [The Zynq Book Tutorials]({{ site.baseurl }}/docs/pdfs/The_Zynq_Book_Tutorials.pdf)
- [Exam 2 Review]({{ site.baseurl }}/docs/pdfs/Exam%202%20Review.pdf)
- [Final Review]({{ site.baseurl }}/ece3623/Final%20Review.docx)

---

## Reading Assignments

1. *The Zynq Book*, System Design chapters
2. Review all previous module materials
3. FreeRTOS best practices documentation

---

## Practice Questions

1. A system has three tasks. Task A (period 10ms, WCET 2ms), Task B (period 20ms, WCET 5ms), Task C (period 50ms, WCET 10ms). Calculate total CPU utilization. Is this schedulable with RMS?

2. Design a task architecture for a temperature monitoring system with: sensor reading, filtering, display update, and alarm handling.

3. What debugging steps would you take if a system randomly freezes every few hours?

4. How would you partition a real-time audio effects system between PS and PL?

5. List five best practices for reliable embedded system design.

---

## Summary

System integration combines all embedded systems concepts into functional products. Successful integration requires careful architecture design, robust software structure, thorough testing, and systematic debugging. The skills developed throughout this course—real-time programming, RTOS, hardware interfacing, signal processing, and HLS—come together to create powerful embedded solutions.

---

## Course Conclusion

Congratulations on completing ECE 3623 Embedded Systems! You now have the knowledge and skills to:

- Design real-time embedded systems
- Program with FreeRTOS
- Interface with analog and digital peripherals
- Implement digital signal processing
- Create hardware accelerators with HLS
- Build complete, reliable embedded solutions

These skills are directly applicable to careers in:
- Embedded Systems Engineering
- IoT Development
- Automotive Electronics
- Medical Devices
- Robotics and Automation
- Consumer Electronics

**Best of luck in your future endeavors!**
