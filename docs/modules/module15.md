---
title: "Module 15: Model-Based Design with Simulink"
layout: default
parent: "Progression 5: Hardware Acceleration"
grand_parent: "Course Modules"
nav_order: 3
---

# Module 15: Model-Based Design with Simulink for Zynq FPGA
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain Model-Based Design (MBD) philosophy for FPGA development
2. Create algorithm models in Simulink for hardware implementation
3. Convert floating-point models to fixed-point for FPGA
4. Generate HDL code using HDL Coder
5. Package Simulink-generated IP for Vivado integration
6. Deploy Simulink algorithms to Zynq Programmable Logic (PL)

---

## Overview

Model-Based Design (MBD) provides a graphical approach to developing algorithms for FPGA implementation. Using MATLAB Simulink with HDL Coder, engineers can design, simulate, and automatically generate hardware description language (HDL) code—all from a visual block diagram environment.

![Algorithm to FPGA Design Flow]({{ site.baseurl }}/docs/imgs/simulink/img-000.png)

---

## Algorithm to FPGA Design Flow

### Traditional vs Model-Based Approaches

The journey from algorithm concept to running FPGA hardware can follow multiple paths:

```
┌─────────────────────────────────────────────────────────────────┐
│              ALGORITHM TO FPGA DESIGN FLOWS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PATH 1: Traditional RTL                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │Algorithm │───►│ Manual   │───►│  Logic   │───►│Bitstream │  │
│  │ Design   │    │Verilog/  │    │Synthesis │    │          │  │
│  │          │    │VHDL      │    │          │    │          │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                  (Weeks/Months)                                  │
│                                                                  │
│  PATH 2: High-Level Synthesis (C/C++)                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │Algorithm │───►│ C/C++    │───►│ Vivado   │───►│   HDL    │  │
│  │ Design   │    │ Code     │    │   HLS    │    │ (Auto)   │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                  (Days/Weeks)                                    │
│                                                                  │
│  PATH 3: Model-Based Design (Simulink)                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │Algorithm │───►│ Simulink │───►│   HDL    │───►│   HDL    │  │
│  │ Design   │    │  Model   │    │  Coder   │    │ (Auto)   │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                  (Hours/Days)                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Design Approach Comparison

| Aspect | Manual RTL | Vivado HLS | HDL Coder (Simulink) |
|--------|-----------|------------|---------------------|
| **Input** | Verilog/VHDL | C/C++ code | Simulink model |
| **Design Style** | Text-based, hardware-centric | Text-based, algorithm-centric | Visual/graphical |
| **Learning Curve** | Steep | Moderate | Gentle |
| **Fixed-point** | Manual implementation | HLS types or manual | Automatic conversion |
| **Verification** | HDL testbench | C testbench | Simulink simulation |
| **Best For** | Hardware experts | Algorithm engineers | Control/DSP engineers |
| **Optimization** | Direct HDL control | HLS pragmas | Block properties/constraints |

![Design Flow Comparison]({{ site.baseurl }}/docs/imgs/simulink/img-002.png)

---

## Model-Based Design Workflow

### Complete MBD Flow for Zynq

```
┌─────────────────────────────────────────────────────────────────┐
│              SIMULINK TO ZYNQ PL WORKFLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────┐                                              │
│  │  1. ALGORITHM │  Design algorithm in Simulink                │
│  │     MODEL     │  (floating-point, behavioral)                │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  2. SIMULATE  │  Verify functional correctness               │
│  │   & VERIFY    │  (compare against golden reference)          │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  3. FIXED-PT  │  Convert to fixed-point representation      │
│  │   CONVERSION  │  (Fixed-Point Designer)                      │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  4. HDL CODE  │  Generate Verilog/VHDL                       │
│  │  GENERATION   │  (HDL Coder)                                 │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  5. IP CORE   │  Package as AXI-compatible IP                │
│  │   PACKAGING   │  (HDL Workflow Advisor)                      │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  6. VIVADO    │  Add IP to block design                      │
│  │  INTEGRATION  │  Connect to Zynq PS                          │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │  7. DEPLOY    │  Generate bitstream, run on Zynq             │
│  │   TO ZYNQ     │                                              │
│  └───────────────┘                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![MBD Workflow]({{ site.baseurl }}/docs/imgs/simulink/img-004.png)

---

## Step 1: Algorithm Design in Simulink

### Hardware-Compatible Blocks

Not all Simulink blocks can be synthesized to hardware. Use blocks from these libraries:

| Library | Block Examples | Hardware Mapping |
|---------|----------------|------------------|
| **Math Operations** | Add, Subtract, Gain, Product | Adders, multipliers |
| **Discrete** | Unit Delay, FIR Filter, IIR Filter | Registers, filter structures |
| **Logic** | AND, OR, NOT, Compare | LUTs, comparators |
| **Signal Routing** | Mux, Demux, Switch | Multiplexers |
| **Sources** | Constant | Fixed values |
| **Sinks** | Scope (simulation only) | - |

![Simulink Hardware Blocks]({{ site.baseurl }}/docs/imgs/simulink/img-006.png)

### Example: Simple Gain Block

```
Input ──►[ × K ]──► Output

In Simulink:
┌─────────┐     ┌─────────┐     ┌─────────┐
│  Input  │────►│  Gain   │────►│ Output  │
│  Port   │     │  (K=5)  │     │  Port   │
└─────────┘     └─────────┘     └─────────┘
```

### Example: FIR Filter Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIR FILTER IN SIMULINK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐    ┌───────────────────┐    ┌─────────┐           │
│   │  Input  │───►│   Discrete FIR    │───►│ Output  │           │
│   │  Port   │    │   Filter Block    │    │  Port   │           │
│   └─────────┘    │                   │    └─────────┘           │
│                  │ Coefficients:     │                          │
│                  │ [0.1 0.2 0.4 0.2  │                          │
│                  │  0.1]             │                          │
│                  └───────────────────┘                          │
│                                                                  │
│   Generates:                                                     │
│   - Shift register chain                                        │
│   - Coefficient multipliers                                     │
│   - Accumulator/adder tree                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 2: Simulation and Verification

### Functional Verification

Before generating hardware, verify the algorithm behaves correctly:

```matlab
% MATLAB script for FIR filter verification
% Define filter coefficients
coeffs = [0.1 0.2 0.4 0.2 0.1];

% Generate test input
t = 0:0.001:1;
input_signal = sin(2*pi*50*t) + 0.5*sin(2*pi*200*t);

% Expected output (MATLAB reference)
expected_output = filter(coeffs, 1, input_signal);

% Run Simulink model
sim('fir_filter_model');

% Compare outputs
error = max(abs(simulink_output - expected_output));
fprintf('Maximum error: %e\n', error);
```

### Test Patterns

| Test Type | Purpose | Input Pattern |
|-----------|---------|---------------|
| Impulse | Verify coefficient values | [1, 0, 0, 0, ...] |
| Step | Check DC gain | [1, 1, 1, 1, ...] |
| Sinusoid | Verify frequency response | sin(2πft) |
| Chirp | Wideband response | Swept frequency |
| Random | General verification | randn() |

![Simulation and Verification]({{ site.baseurl }}/docs/imgs/simulink/img-008.png)

---

## Step 3: Fixed-Point Conversion

### Why Fixed-Point?

FPGAs excel at fixed-point arithmetic. Floating-point requires significantly more resources:

| Operation | Fixed-Point (16-bit) | Floating-Point (32-bit) |
|-----------|---------------------|------------------------|
| Multiplier | 1 DSP slice | 3-4 DSP slices |
| Adder | ~16 LUTs | ~500 LUTs |
| Latency | 1-3 cycles | 5-10 cycles |

### Fixed-Point Representation

A fixed-point number uses a specified word length and fraction length:

$$\text{Value} = \frac{\text{Integer Representation}}{2^{FL}}$$

Where:
- $WL$ = Word Length (total bits)
- $FL$ = Fraction Length (bits after binary point)
- Range: $[-2^{WL-FL-1}, 2^{WL-FL-1} - 2^{-FL}]$

**Example**: 16-bit signed, 14 fraction bits (Q1.14)
- Range: [-2, 2 - 2^-14] ≈ [-2, 1.99994]
- Resolution: $2^{-14}$ ≈ 0.000061

### Fixed-Point Designer Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│              FIXED-POINT CONVERSION PROCESS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Instrument Model                                            │
│     - Add "Data Type Override" block                            │
│     - Run simulation to collect signal ranges                   │
│                                                                  │
│  2. Analyze Ranges                                              │
│     - Fixed-Point Tool shows min/max values                     │
│     - Identifies overflow potential                             │
│                                                                  │
│  3. Propose Word Lengths                                        │
│     - Automatic proposal based on ranges                        │
│     - User can set word length constraints                      │
│                                                                  │
│  4. Apply Fixed-Point Types                                     │
│     - Convert all blocks to fixed-point                         │
│     - Verify quantization error acceptable                      │
│                                                                  │
│  5. Validate                                                    │
│     - Compare fixed-point vs floating-point outputs             │
│     - Ensure error within specification                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![Fixed-Point Conversion]({{ site.baseurl }}/docs/imgs/simulink/img-010.png)

### Simulink Fixed-Point Settings

```matlab
% Set fixed-point properties for a Gain block
set_param('model/Gain', 'OutDataTypeStr', 'fixdt(1,16,14)');
%                                          ↑  ↑   ↑
%                                     signed  WL  FL

% Common fixed-point specifications:
% fixdt(1, 16, 15)  - Q0.15: Range [-1, 1)
% fixdt(1, 32, 16)  - Q15.16: High precision
% fixdt(0, 12, 0)   - Unsigned integer (ADC values)
```

---

## Step 4: HDL Code Generation

### HDL Coder Overview

HDL Coder converts Simulink models to synthesizable Verilog or VHDL:

```
┌─────────────────────────────────────────────────────────────────┐
│                    HDL CODER PROCESS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Simulink Model                                                 │
│        │                                                         │
│        ▼                                                         │
│  ┌───────────────────┐                                          │
│  │ HDL Compatibility │  Check blocks are HDL-compatible         │
│  │     Checker       │                                          │
│  └─────────┬─────────┘                                          │
│            │                                                     │
│            ▼                                                     │
│  ┌───────────────────┐                                          │
│  │  Code Generation  │  Apply architecture optimizations        │
│  │     Settings      │  (pipelining, sharing, streaming)        │
│  └─────────┬─────────┘                                          │
│            │                                                     │
│            ▼                                                     │
│  ┌───────────────────┐                                          │
│  │ Generate HDL Code │  Output Verilog or VHDL files            │
│  └─────────┬─────────┘                                          │
│            │                                                     │
│            ▼                                                     │
│  ┌───────────────────┐                                          │
│  │ Generate Testbench│  Create HDL testbench for verification   │
│  └───────────────────┘                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![HDL Coder Process]({{ site.baseurl }}/docs/imgs/simulink/img-012.png)

### HDL Coder Settings

```matlab
% Create HDL Coder configuration
hdlcfg = coder.config('hdl');

% Target language
hdlcfg.TargetLanguage = 'Verilog';  % or 'VHDL'

% Clock settings
hdlcfg.TargetFrequency = 100;  % 100 MHz

% Optimization
hdlcfg.ShareMathOperations = 'on';  % Share multipliers
hdlcfg.DistributedPipelining = 'on';  % Add pipeline registers

% Generate code
makehdl('fir_filter_subsystem');
```

### Generated HDL Structure

```verilog
// Auto-generated by HDL Coder
module fir_filter
  (
   input  clk,
   input  reset,
   input  clk_enable,
   input  signed [15:0] data_in,
   output signed [31:0] data_out,
   output ce_out
  );

  // Coefficient constants
  parameter signed [15:0] coeff1 = 16'b0000110011001100;  // 0.1
  parameter signed [15:0] coeff2 = 16'b0001100110011001;  // 0.2
  // ...

  // Delay registers
  reg signed [15:0] delay_pipeline [0:4];

  // Filter implementation
  always @(posedge clk) begin
    if (reset) begin
      // Reset logic
    end else if (clk_enable) begin
      // Shift register and MAC operations
    end
  end

endmodule
```

![Generated HDL Code]({{ site.baseurl }}/docs/imgs/simulink/img-014.png)

---

## Step 5: IP Core Generation

### HDL Workflow Advisor

The HDL Workflow Advisor automates the process of packaging Simulink-generated HDL as a Vivado IP:

```
┌─────────────────────────────────────────────────────────────────┐
│                HDL WORKFLOW ADVISOR STEPS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Set Target                                             │
│    └─► Select "Xilinx Zynq Platform"                            │
│    └─► Choose board (e.g., "Zybo Z7")                           │
│                                                                  │
│  Step 2: Set Target Reference Design                            │
│    └─► Default system (AXI4-Lite interface)                     │
│    └─► Or custom reference design                               │
│                                                                  │
│  Step 3: Set Target Interface                                   │
│    └─► Map Simulink ports to AXI registers                      │
│    └─► Configure AXI4-Stream for streaming data                 │
│                                                                  │
│  Step 4: Generate RTL Code and IP Core                          │
│    └─► Creates packaged IP for Vivado                           │
│    └─► Generates software driver files                          │
│                                                                  │
│  Step 5: Build FPGA Bitstream                                   │
│    └─► Runs Vivado synthesis and implementation                 │
│    └─► Generates .bit file                                      │
│                                                                  │
│  Step 6: Program Target Device                                  │
│    └─► Download to Zynq board                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AXI Interface Options

| Interface Type | Use Case | Bandwidth |
|---------------|----------|-----------|
| **AXI4-Lite** | Control registers, parameters | Low (words) |
| **AXI4** | Memory-mapped data | High (bursts) |
| **AXI4-Stream** | Continuous data streaming | Highest |

### Port Mapping Example

```
Simulink Model:                    AXI Interface:
┌─────────────────┐               ┌─────────────────┐
│                 │               │                 │
│  Inport 1 ──────┼──────────────►│ AXI4-Lite Reg 0 │
│  (gain value)   │               │ (software write)│
│                 │               │                 │
│  Inport 2 ──────┼──────────────►│ AXI4-Stream In  │
│  (data stream)  │               │ (DMA input)     │
│                 │               │                 │
│  Outport 1 ─────┼──────────────►│ AXI4-Stream Out │
│  (filtered data)│               │ (DMA output)    │
│                 │               │                 │
└─────────────────┘               └─────────────────┘
```

![IP Core Generation]({{ site.baseurl }}/docs/imgs/simulink/img-016.png)

---

## Step 6: Vivado Integration

### Adding Simulink IP to Block Design

After HDL Workflow Advisor generates the IP:

```
┌─────────────────────────────────────────────────────────────────┐
│                    VIVADO BLOCK DESIGN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     ZYNQ PS                              │   │
│   │  ┌──────────────────────────────────────────────────┐   │   │
│   │  │           ARM Cortex-A9 Processors               │   │   │
│   │  └──────────────────────────────────────────────────┘   │   │
│   │            │ M_AXI_GP0          │ S_AXI_HP0             │   │
│   └────────────┼────────────────────┼───────────────────────┘   │
│                │                    │                            │
│                ▼                    ▼                            │
│   ┌────────────────────┐    ┌────────────────────┐              │
│   │  AXI Interconnect  │    │    AXI DMA         │              │
│   └─────────┬──────────┘    └─────────┬──────────┘              │
│             │                         │                          │
│             ▼                         ▼                          │
│   ┌────────────────────────────────────────────────────┐        │
│   │           SIMULINK-GENERATED IP CORE               │        │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │        │
│   │  │ AXI4-Lite   │  │   Filter    │  │ AXI4-Stream │ │        │
│   │  │ Interface   │──│   Logic     │──│  Interface  │ │        │
│   │  │ (control)   │  │             │  │  (data)     │ │        │
│   │  └─────────────┘  └─────────────┘  └─────────────┘ │        │
│   └────────────────────────────────────────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![Vivado Block Design]({{ site.baseurl }}/docs/imgs/simulink/img-018.png)

---

## Step 7: Software Driver and Deployment

### Generated Driver Files

HDL Coder generates C driver files for controlling the IP from software:

```c
/* Auto-generated by HDL Coder */
#include "fir_filter_ip.h"

// Register offsets
#define FILTER_CTRL_REG     0x00
#define FILTER_STATUS_REG   0x04
#define FILTER_COEFF_REG    0x08
#define FILTER_GAIN_REG     0x0C

// Initialize IP core
void FIR_Filter_Init(uint32_t baseAddr) {
    // Write default gain
    Xil_Out32(baseAddr + FILTER_GAIN_REG, 0x4000);  // 1.0 in Q1.14
    
    // Enable filter
    Xil_Out32(baseAddr + FILTER_CTRL_REG, 0x01);
}

// Set filter gain (Q1.14 format)
void FIR_Filter_SetGain(uint32_t baseAddr, uint16_t gain) {
    Xil_Out32(baseAddr + FILTER_GAIN_REG, gain);
}

// Check if filter is ready
uint32_t FIR_Filter_IsReady(uint32_t baseAddr) {
    return (Xil_In32(baseAddr + FILTER_STATUS_REG) & 0x01);
}
```

### FreeRTOS Integration

```c
#include "FreeRTOS.h"
#include "task.h"
#include "fir_filter_ip.h"

#define FILTER_BASEADDR  XPAR_FIR_FILTER_IP_0_BASEADDR

void vFilterControlTask(void *pvParameters) {
    // Initialize Simulink-generated IP
    FIR_Filter_Init(FILTER_BASEADDR);
    
    for (;;) {
        // Adjust filter gain based on system requirements
        uint16_t gain = CalculateGain();
        FIR_Filter_SetGain(FILTER_BASEADDR, gain);
        
        // Wait before next update
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

![Software Driver Integration]({{ site.baseurl }}/docs/imgs/simulink/img-020.png)

---

## Complete Example: Audio Filter

### System Requirements

- **Input**: 16-bit audio from ADC at 48 kHz
- **Processing**: Configurable low-pass filter
- **Output**: Filtered audio to DAC

### Simulink Model

```
┌─────────────────────────────────────────────────────────────────┐
│                  AUDIO FILTER SIMULINK MODEL                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Audio   │    │ FIR      │    │  Gain    │    │  Audio   │  │
│  │  Input   │───►│ Filter   │───►│  Stage   │───►│ Output   │  │
│  │ (AXI-S)  │    │ (32 tap) │    │  (var)   │    │ (AXI-S)  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                        ▲              ▲                          │
│                        │              │                          │
│                  ┌─────┴────┐   ┌─────┴────┐                    │
│                  │ Coeff    │   │  Gain    │                    │
│                  │ Select   │   │  Value   │                    │
│                  │(AXI-Lite)│   │(AXI-Lite)│                    │
│                  └──────────┘   └──────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Performance Results

| Metric | Floating-Point Model | Fixed-Point FPGA |
|--------|---------------------|------------------|
| Latency | N/A (simulation) | 35 clock cycles |
| Throughput | N/A | 48,000 samples/sec |
| Resources | N/A | 2 DSP, 450 LUT, 380 FF |
| Power | N/A | ~50 mW |

![Audio Filter Example]({{ site.baseurl }}/docs/imgs/simulink/img-022.png)

---

## Simulink vs Vivado HLS Comparison

### When to Use Each Approach

| Scenario | Recommended Tool |
|----------|-----------------|
| Algorithm involves control systems | Simulink + HDL Coder |
| Team has MATLAB/Simulink experience | Simulink + HDL Coder |
| Complex fixed-point analysis needed | Simulink + Fixed-Point Designer |
| Team has C/C++ experience | Vivado HLS |
| Need fine-grained HW control | Vivado HLS |
| Algorithm is data-flow oriented | Either works well |

### Feature Comparison

| Feature | Simulink/HDL Coder | Vivado HLS |
|---------|-------------------|------------|
| Input format | Block diagram | C/C++ code |
| Visualization | Built-in scopes | Printf/waveform viewer |
| Fixed-point conversion | Semi-automatic | Manual/HLS types |
| Optimization control | Block properties | Pragmas |
| Learning curve | Lower for DSP | Lower for SW engineers |
| Cost | MATLAB license required | Included in Vivado |

![Comparison Summary]({{ site.baseurl }}/docs/imgs/simulink/img-024.png)

---

## Xilinx SoC Blockset

### Enhanced PS-PL Integration

The **Xilinx SoC Blockset** provides additional blocks for tight Zynq integration:

| Block | Function |
|-------|----------|
| **Zynq Hardware** | Model Zynq PS for simulation |
| **AXI4-Stream Read/Write** | DMA-style data transfer |
| **AXI Manager** | Memory-mapped access |
| **External IO** | GPIO and peripheral access |

```
┌─────────────────────────────────────────────────────────────────┐
│              XILINX SOC BLOCKSET INTEGRATION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   MATLAB/Simulink                    Zynq Target                │
│   ┌─────────────────────┐           ┌─────────────────────┐     │
│   │ Processor Model     │           │    ARM Cortex-A9    │     │
│   │  (software)         │◄─────────►│    (runs software)  │     │
│   └─────────────────────┘           └─────────────────────┘     │
│            │                                  │                  │
│            │ AXI Interface                    │ AXI              │
│            ▼                                  ▼                  │
│   ┌─────────────────────┐           ┌─────────────────────┐     │
│   │ FPGA Algorithm      │           │   PL Hardware       │     │
│   │  (hardware model)   │──────────►│  (generated HDL)    │     │
│   └─────────────────────┘           └─────────────────────┘     │
│                                                                  │
│   Enables: Hardware-in-the-Loop verification                    │
│            External mode for live parameter tuning              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![Xilinx SoC Blockset]({{ site.baseurl }}/docs/imgs/simulink/img-026.png)

---

## Lecture Materials

- [Algorithm to FPGA Hardware Design Flow]({{ site.baseurl }}/docs/pdfs/Algorithm_to_FPGA_Hardware_Design_Flow.pdf)
- [Week 12 HLS Slides]({{ site.baseurl }}/docs/pdfs/Week%2012%20M-W-HLS.pptx)
- [The Zynq Book Tutorials]({{ site.baseurl }}/docs/pdfs/The_Zynq_Book_Tutorials.pdf)

---

## Reading Assignments

1. MathWorks HDL Coder Documentation: Getting Started
2. Xilinx SoC Blockset User Guide
3. "Model-Based Design for Embedded Systems" (Madhavan)

---

## Practice Questions

1. What are the three main paths from algorithm to FPGA hardware? Compare their advantages.

2. Why is fixed-point conversion necessary for FPGA implementation? Calculate the range and resolution of a fixdt(1, 16, 12) number.

3. A Simulink model uses a 32-tap FIR filter at 100 MHz. If HDL Coder generates a fully pipelined design with II=1, what is the maximum sample rate?

4. Explain the difference between AXI4-Lite and AXI4-Stream interfaces. When would you use each?

5. You have a control algorithm in Simulink that runs at 10 kHz. Describe the steps to deploy it to the Zybo Z7 programmable logic.

---

## Summary

Model-Based Design with Simulink provides a powerful alternative to traditional HDL coding or C-based HLS. Key advantages include:

- **Visual design**: Block diagrams are intuitive for DSP and control engineers
- **Automatic fixed-point conversion**: Reduces error-prone manual conversion
- **Simulation before synthesis**: Verify behavior before committing to hardware
- **Integrated workflow**: From algorithm to deployed FPGA in one environment
- **IP packaging**: Seamless integration with Vivado and Zynq PS

The choice between Simulink/HDL Coder and Vivado HLS depends on team expertise, project requirements, and available licenses. Both tools ultimately generate HDL that can be integrated into Zynq designs.

