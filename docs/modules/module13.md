---
title: "Module 13: High-Level Synthesis"
layout: default
parent: "Progression 5: Hardware Acceleration"
grand_parent: "Course Modules"
nav_order: 1
---

# Module 13: High-Level Synthesis (HLS) Design Flow
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain how Vivado HLS extracts hardware from C code
2. Understand the scheduling process (determining clock cycles)
3. Understand the binding process (assigning library cells)
4. Perform C validation to verify algorithm correctness
5. Perform RTL verification to confirm hardware matches C behavior

---

## Overview

High-Level Synthesis (HLS) transforms C/C++ algorithms into hardware description language (HDL), dramatically accelerating FPGA design time. This module details the complete HLS flow from algorithm to verified hardware IP.

---

## Why High-Level Synthesis?

### Traditional RTL Design Flow

```
Algorithm     Manual HDL      Synthesis    Place &     Bitstream
  Design  ──► Coding     ──►   (Logic)  ──► Route  ──►  
           (Weeks/Months)
```

### HLS Design Flow

```
Algorithm     C/C++       Vivado      HDL        Synthesis   Bitstream
  Design  ──► Code   ──►  HLS    ──► (Auto)  ──►  (Logic) ──►
           (Hours/Days)
```

### Benefits of HLS

| Benefit | Description |
|---------|-------------|
| **Faster Development** | C/C++ is faster to write and debug than HDL |
| **Algorithm Focus** | Focus on what to compute, not how to clock it |
| **Design Exploration** | Quickly try different implementations |
| **Easier Verification** | Test with C before hardware synthesis |
| **Portability** | Same C code can target different FPGAs |

---

## The HLS Design Flow

### Complete Flow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    HLS DESIGN FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐                │
│  │  C/C++    │    │    C      │    │   Pass?   │                │
│  │  Source   │───►│Simulation │───►│   Yes/No  │                │
│  │   Code    │    │(Testbench)│    │           │                │
│  └───────────┘    └───────────┘    └─────┬─────┘                │
│       │                                  │ Yes                   │
│       │                                  ▼                       │
│       │           ┌──────────────────────────────┐               │
│       └──────────►│      C SYNTHESIS             │               │
│                   │  ┌────────────────────────┐  │               │
│                   │  │     SCHEDULING         │  │               │
│                   │  │ (Assign ops to cycles) │  │               │
│                   │  └──────────┬─────────────┘  │               │
│                   │             ▼                │               │
│                   │  ┌────────────────────────┐  │               │
│                   │  │      BINDING           │  │               │
│                   │  │ (Assign ops to HW)     │  │               │
│                   │  └──────────┬─────────────┘  │               │
│                   │             ▼                │               │
│                   │  ┌────────────────────────┐  │               │
│                   │  │    RTL GENERATION      │  │               │
│                   │  │ (Verilog/VHDL output)  │  │               │
│                   │  └────────────────────────┘  │               │
│                   └──────────────┬───────────────┘               │
│                                  ▼                               │
│                   ┌──────────────────────────────┐               │
│                   │    C/RTL CO-SIMULATION       │               │
│                   │  (Verify RTL matches C)      │               │
│                   └──────────────┬───────────────┘               │
│                                  ▼                               │
│                   ┌──────────────────────────────┐               │
│                   │       EXPORT RTL/IP          │               │
│                   │   (Package for Vivado)       │               │
│                   └──────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: C Validation

### Purpose

Verify the algorithm is **functionally correct** before synthesis.

### Testbench Structure

```c
// testbench.c
#include <stdio.h>
#include "myfilter.h"

int main() {
    int input[100];
    int output[100];
    int expected[100];
    int errors = 0;
    
    // Generate test input
    for (int i = 0; i < 100; i++) {
        input[i] = i * 10;
        expected[i] = compute_expected(input[i]);  // Golden model
    }
    
    // Call DUT (Design Under Test)
    myfilter(input, output, 100);
    
    // Compare results
    for (int i = 0; i < 100; i++) {
        if (output[i] != expected[i]) {
            printf("ERROR at %d: got %d, expected %d\n", 
                   i, output[i], expected[i]);
            errors++;
        }
    }
    
    if (errors == 0) {
        printf("TEST PASSED!\n");
        return 0;
    } else {
        printf("TEST FAILED: %d errors\n", errors);
        return 1;
    }
}
```

### Design Under Test (DUT)

```c
// myfilter.c
#include "myfilter.h"

void myfilter(int input[], int output[], int n) {
    for (int i = 0; i < n; i++) {
        // Simple gain and offset
        output[i] = (input[i] * 3) + 100;
    }
}
```

---

## Step 2: C Synthesis - Scheduling

### What is Scheduling?

**Scheduling** assigns each operation to a specific clock cycle.

### Example: Without Pipelining

```c
int result = (a + b) * c + d;
```

**Scheduled operations:**

```
Cycle 1: temp1 = a + b      [ADD]
Cycle 2: temp2 = temp1 * c  [MUL]
Cycle 3: result = temp2 + d [ADD]

Total Latency: 3 cycles
```

### Scheduling Diagram

```
Clock:    │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │...
──────────┼───┼───┼───┼───┼───┼───┼───
Add(a,b)  │███│   │   │   │   │   │
Mul(t1,c) │   │███│   │   │   │   │
Add(t2,d) │   │   │███│   │   │   │
Output    │   │   │   │OUT│   │   │

Latency = 3 cycles (input to output)
```

### Resource Constraints Affect Scheduling

If only ONE adder is available:

```c
int result = a + b + c + d;
```

```
Cycle 1: t1 = a + b  [ADD]
Cycle 2: t2 = t1 + c [ADD]
Cycle 3: result = t2 + d [ADD]

With 2 adders:
Cycle 1: t1 = a + b [ADD1], t2 = c + d [ADD2]
Cycle 2: result = t1 + t2 [ADD1]
```

---

## Step 3: C Synthesis - Binding

### What is Binding?

**Binding** assigns scheduled operations to specific hardware resources (library cells).

### Library Cells

| Operation | Possible Implementations |
|-----------|-------------------------|
| Addition | Ripple-carry adder, Carry-lookahead adder |
| Multiplication | Array multiplier, DSP slice |
| Memory | BRAM, LUT RAM, Distributed RAM |
| Register | Flip-flops |

### Binding Example

```
┌─────────────────────────────────────────────────────────────────┐
│                    BINDING DECISIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Operation          Library Cell          FPGA Resource        │
│   ─────────          ────────────          ─────────────        │
│   16-bit multiply    DSP48 slice           1 DSP                │
│   32-bit add         LUT-based adder       32 LUTs              │
│   Array[1024]        Block RAM             1 BRAM               │
│   Register           Flip-flop             8 FFs                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Binding Visualization

```
    C Code                    Hardware
    ───────                   ────────
    
    a * b         ───────►    ┌─────────────┐
                              │   DSP48E1   │
                              │  Multiplier │
                              └─────────────┘
    
    c + d         ───────►    ┌─────────────┐
                              │  LUT Adder  │
                              │  (32 LUTs)  │
                              └─────────────┘
    
    int arr[256]  ───────►    ┌─────────────┐
                              │    BRAM     │
                              │  (1 block)  │
                              └─────────────┘
```

---

## Step 4: RTL Generation

### Output Files

HLS generates synthesizable HDL:

```
project/
├── solution1/
│   ├── syn/
│   │   ├── verilog/
│   │   │   ├── myfilter.v      ← Verilog implementation
│   │   │   └── myfilter_ctrl.v
│   │   └── vhdl/
│   │       └── myfilter.vhd    ← VHDL implementation
│   └── impl/
│       └── ip/
│           └── myfilter.zip    ← Packaged IP for Vivado
```

### Generated Interface

```verilog
module myfilter (
    input  wire        ap_clk,
    input  wire        ap_rst,
    input  wire        ap_start,
    output wire        ap_done,
    output wire        ap_idle,
    output wire        ap_ready,
    input  wire [31:0] input_r,
    output wire [31:0] output_r
);
```

---

## Step 5: C/RTL Co-Simulation

### Purpose

Verify that the generated RTL **behaves identically** to the original C code.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    CO-SIMULATION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐         ┌──────────────┐                     │
│   │  C Testbench │         │  C Testbench │                     │
│   │   (input)    │         │   (verify)   │                     │
│   └──────┬───────┘         └──────▲───────┘                     │
│          │                        │                              │
│          ▼                        │                              │
│   ┌──────────────┐         ┌──────┴───────┐                     │
│   │   Wrapper    │────────►│   Wrapper    │                     │
│   │  (C → RTL)   │         │  (RTL → C)   │                     │
│   └──────┬───────┘         └──────▲───────┘                     │
│          │                        │                              │
│          ▼                        │                              │
│   ┌────────────────────────────────────────┐                    │
│   │         RTL Simulator (XSIM)           │                    │
│   │         Simulates Verilog/VHDL         │                    │
│   └────────────────────────────────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Verification Report

```
================================================================
== Co-simulation Results
================================================================
RTL Simulation status: PASS
Latency: 5 cycles
Interval: 1 cycle
Errors: 0

Clock period: 10 ns
Total simulation time: 1000 ns
```

---

## Synthesis Reports

### Key Metrics

| Metric | Description |
|--------|-------------|
| **Latency** | Clock cycles from first input to first output |
| **Interval (II)** | Clock cycles between consecutive inputs |
| **BRAM** | Block RAM utilization |
| **DSP** | DSP slice utilization |
| **FF** | Flip-flop count |
| **LUT** | Look-up table count |

### Example Synthesis Report

```
================================================================
== Performance Estimates
================================================================
+ Timing (ns):
    * Summary:
        +--------+-------+----------+------------+
        |  Clock | Target| Estimated| Uncertainty|
        +--------+-------+----------+------------+
        |ap_clk  |  10.00|     7.23 |    1.25    |
        +--------+-------+----------+------------+

+ Latency (clock cycles):
    * Summary:
        +-----+-----+-----+-----+---------+
        |  Latency  |  Interval | Pipeline|
        | min | max | min | max |   Type  |
        +-----+-----+-----+-----+---------+
        |   10|   10|   10|   10|   none  |
        +-----+-----+-----+-----+---------+

================================================================
== Utilization Estimates
================================================================
+-----------------+---------+-------+-------+-------+
|       Name      | BRAM_18K| DSP48E|   FF  |  LUT  |
+-----------------+---------+-------+-------+-------+
|Expression       |        -|      2|      0|     64|
|FIFO             |        -|      -|      -|      -|
|Instance         |        -|      -|      -|      -|
|Memory           |        1|      -|      -|      -|
|Multiplexer      |        -|      -|      -|     32|
|Register         |        -|      -|     48|      -|
+-----------------+---------+-------+-------+-------+
|Total            |        1|      2|     48|     96|
+-----------------+---------+-------+-------+-------+
```

---

## Complete HLS Example: FIR Filter

### C Source

```c
// fir.h
#ifndef FIR_H
#define FIR_H

#define N 16
typedef int coef_t;
typedef int data_t;
typedef int acc_t;

void fir(data_t *y, data_t x);

#endif
```

```c
// fir.c
#include "fir.h"

void fir(data_t *y, data_t x) {
    static data_t shift_reg[N];
    static const coef_t c[N] = {
        1, 2, 3, 4, 5, 6, 7, 8,
        8, 7, 6, 5, 4, 3, 2, 1
    };
    acc_t acc = 0;
    int i;
    
    // Shift register
    Shift_Loop:
    for (i = N-1; i > 0; i--) {
        shift_reg[i] = shift_reg[i-1];
    }
    shift_reg[0] = x;
    
    // MAC operations
    MAC_Loop:
    for (i = 0; i < N; i++) {
        acc += shift_reg[i] * c[i];
    }
    
    *y = acc;
}
```

### Testbench

```c
// fir_tb.c
#include <stdio.h>
#include "fir.h"

int main() {
    data_t y;
    data_t test_input[20] = {1,2,3,4,5,6,7,8,9,10,
                              10,9,8,7,6,5,4,3,2,1};
    
    printf("FIR Filter Test\n");
    
    for (int i = 0; i < 20; i++) {
        fir(&y, test_input[i]);
        printf("Input: %d, Output: %d\n", test_input[i], y);
    }
    
    return 0;
}
```

---

## Lecture Materials

- [Week 13 - HLS Introduction]({{ site.baseurl }}/ece3623/Week%2013%20HLS.pdf)
- [Vivado HLS Tutorial]({{ site.baseurl }}/ece3623/Vivado_HLS_Tutorial.pdf)
- [HLS Design Flow]({{ site.baseurl }}/ece3623/HLS%20Design%20Flow.pdf)

---

## Reading Assignments

1. Xilinx UG1399: Vitis HLS User Guide, Chapters 1-3
2. *The Zynq Book*, Chapter 14: High-Level Synthesis
3. HLS design methodology papers

---

## Practice Questions

1. What are the two main steps in C synthesis (hint: scheduling and ___)?

2. What does scheduling determine in the HLS flow?

3. What does binding determine in the HLS flow?

4. Why is C validation performed before synthesis?

5. What is the purpose of C/RTL co-simulation?

6. An operation has latency=20 cycles and II=1. How many cycles to process 100 samples?

7. If only one multiplier is available, how does this affect scheduling of `a*b + c*d`?

---

## Summary

Vivado HLS transforms C/C++ algorithms into hardware by: (1) validating the algorithm through C simulation, (2) scheduling operations to clock cycles, (3) binding operations to hardware resources, (4) generating RTL, and (5) verifying correctness through co-simulation. This automated flow significantly accelerates FPGA development while allowing designers to focus on algorithms rather than timing details.

---

## Next Module

[Module 14: HLS Directives and Optimization →]({{ site.baseurl }}/docs/modules/module14.html)
}
```

### Array Processing Example

```c
#define N 1024

void array_sum(int input[N], int *sum)
{
    int temp = 0;
    
    for (int i = 0; i < N; i++) {
        temp += input[i];
    }
    
    *sum = temp;
}
```

### FIR Filter in HLS

```c
#define TAPS 16

typedef int coef_t;
typedef int data_t;
typedef int acc_t;

void fir_filter(data_t input, data_t *output, coef_t coeffs[TAPS])
{
    static data_t shift_reg[TAPS] = {0};
    acc_t acc = 0;
    
    /* Shift register */
    for (int i = TAPS - 1; i > 0; i--) {
        shift_reg[i] = shift_reg[i-1];
    }
    shift_reg[0] = input;
    
    /* MAC operations */
    for (int i = 0; i < TAPS; i++) {
        acc += shift_reg[i] * coeffs[i];
    }
    
    *output = acc;
}
```

---

## HLS Pragmas (Optimization Directives)

Pragmas guide the HLS tool to generate optimized hardware.

### Pipeline

Execute loop iterations concurrently with specified initiation interval.

```c
void pipelined_sum(int input[N], int *sum)
{
    int temp = 0;
    
    #pragma HLS PIPELINE II=1
    for (int i = 0; i < N; i++) {
        temp += input[i];
    }
    
    *sum = temp;
}
```

### Unroll

Replicate loop body to increase parallelism.

```c
void unrolled_mac(int a[8], int b[8], int *result)
{
    int sum = 0;
    
    #pragma HLS UNROLL factor=4
    for (int i = 0; i < 8; i++) {
        sum += a[i] * b[i];
    }
    
    *result = sum;
}
```

### Array Partition

Split arrays into smaller pieces for parallel access.

```c
void parallel_access(int input[N], int output[N])
{
    #pragma HLS ARRAY_PARTITION variable=input complete
    #pragma HLS ARRAY_PARTITION variable=output complete
    
    for (int i = 0; i < N; i++) {
        #pragma HLS UNROLL
        output[i] = input[i] * 2;
    }
}
```

### Interface Pragmas

Define how the function interfaces with external systems.

```c
void axi_stream_example(
    hls::stream<int> &input_stream,
    hls::stream<int> &output_stream)
{
    #pragma HLS INTERFACE axis port=input_stream
    #pragma HLS INTERFACE axis port=output_stream
    #pragma HLS INTERFACE ap_ctrl_none port=return
    
    int data = input_stream.read();
    output_stream.write(data * 2);
}
```

---

## Common Pragmas Reference

| Pragma | Purpose |
|--------|---------|
| `PIPELINE` | Pipeline loops/functions |
| `UNROLL` | Unroll loops |
| `ARRAY_PARTITION` | Split arrays |
| `DATAFLOW` | Task-level pipelining |
| `INTERFACE` | Define I/O protocols |
| `INLINE` | Inline functions |
| `LATENCY` | Constrain latency |
| `RESOURCE` | Specify implementation |

---

## AXI Interfaces

### AXI4-Lite (Control/Status)

For scalar arguments and control registers.

```c
#pragma HLS INTERFACE s_axilite port=return bundle=CTRL
#pragma HLS INTERFACE s_axilite port=param bundle=CTRL
```

### AXI4-Stream (Streaming Data)

For continuous data flow.

```c
#pragma HLS INTERFACE axis port=input_data
#pragma HLS INTERFACE axis port=output_data
```

### AXI4 Memory-Mapped (Bulk Data)

For accessing DDR memory.

```c
#pragma HLS INTERFACE m_axi port=data offset=slave bundle=DATA
```

---

## Complete HLS Example: Image Filter

```c
#include <hls_stream.h>
#include <ap_int.h>

#define WIDTH  640
#define HEIGHT 480

typedef ap_uint<8> pixel_t;

void sobel_filter(
    hls::stream<pixel_t> &input,
    hls::stream<pixel_t> &output)
{
    #pragma HLS INTERFACE axis port=input
    #pragma HLS INTERFACE axis port=output
    #pragma HLS INTERFACE ap_ctrl_none port=return
    
    /* Line buffers */
    pixel_t line_buf[2][WIDTH];
    #pragma HLS ARRAY_PARTITION variable=line_buf complete dim=1
    
    pixel_t window[3][3];
    #pragma HLS ARRAY_PARTITION variable=window complete
    
    for (int row = 0; row < HEIGHT; row++) {
        for (int col = 0; col < WIDTH; col++) {
            #pragma HLS PIPELINE II=1
            
            /* Read new pixel */
            pixel_t new_pixel = input.read();
            
            /* Update line buffers and window */
            /* ... (sliding window logic) ... */
            
            /* Apply Sobel operator */
            int gx = window[0][2] - window[0][0] +
                     2*window[1][2] - 2*window[1][0] +
                     window[2][2] - window[2][0];
                     
            int gy = window[2][0] - window[0][0] +
                     2*window[2][1] - 2*window[0][1] +
                     window[2][2] - window[0][2];
            
            int magnitude = (gx > 0 ? gx : -gx) + 
                           (gy > 0 ? gy : -gy);
            
            pixel_t result = (magnitude > 255) ? 255 : magnitude;
            output.write(result);
        }
    }
}
```

---

## Integrating HLS IP with Zynq

### Vivado Integration

1. Export HLS IP to Vivado IP catalog
2. Add to block design
3. Connect AXI interfaces to Zynq PS
4. Generate bitstream

### Software Driver

```c
#include "xfir_filter.h"  /* Generated by HLS */

XFir_filter FilterInstance;

int main()
{
    /* Initialize HLS IP */
    XFir_filter_Initialize(&FilterInstance, XPAR_FIR_FILTER_0_DEVICE_ID);
    
    /* Set parameters via AXI-Lite */
    XFir_filter_Set_coeffs(&FilterInstance, coefficients);
    
    /* Start processing */
    XFir_filter_Start(&FilterInstance);
    
    /* Wait for completion */
    while (!XFir_filter_IsDone(&FilterInstance));
    
    /* Read results */
    int result = XFir_filter_Get_output(&FilterInstance);
    
    return 0;
}
```

---

## Performance Analysis

### Synthesis Report Metrics

| Metric | Description |
|--------|-------------|
| **Latency** | Clock cycles from input to output |
| **Interval (II)** | Cycles between consecutive inputs |
| **BRAM** | Block RAM usage |
| **DSP** | DSP slice usage |
| **FF** | Flip-flop usage |
| **LUT** | Look-up table usage |

### Optimization Goals

- **Throughput**: Minimize II (initiation interval)
- **Latency**: Minimize total clock cycles
- **Resources**: Minimize FPGA utilization
- **Power**: Reduce switching activity

---

## Lecture Materials

- [Week 12 - HLS]({{ site.baseurl }}/ece3623/Week%2012%20M-W-HLS.pptx)
- [HLS Summary]({{ site.baseurl }}/ece3623/HLS%20Summary.docx)

---

## Reading Assignments

1. Xilinx UG1399: Vitis HLS User Guide (selected chapters)
2. *The Zynq Book*, Chapter on HLS
3. HLS optimization tutorials

---

## Practice Questions

1. What are the main advantages of using HLS over traditional HDL design?

2. What does `#pragma HLS PIPELINE II=1` achieve?

3. Explain the difference between loop unrolling and pipelining.

4. Why might you use array partitioning?

5. An HLS function has latency=100 and II=1. How many clock cycles to process 1000 samples?

---

## Summary

High-Level Synthesis bridges software algorithms and hardware implementation, dramatically reducing development time. Through pragmas and careful code structure, you can create efficient hardware accelerators from C/C++ code. Combined with Zynq's ARM processor, HLS enables powerful heterogeneous computing solutions. The final module brings everything together in complete system integration.

---

## Next Module

[Module 14: System Integration →]({{ site.baseurl }}/docs/modules/module14.html)
