---
title: "Lab 7: Digital Filters"
layout: default
parent: Laboratory Exercises
nav_order: 7
---

# Lab 7: Digital Filters
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will implement digital filters on the Zynq processor to process real-time audio/signal data. You will design and implement FIR (Finite Impulse Response) and IIR (Infinite Impulse Response) filters, then test them with the PmodAD1 and PmodDA2 modules.

**Related Modules**: [Module 10: Analog-to-Digital Conversion](../modules/module10/), [Module 12: Digital Signal Processing](../modules/module12/)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Implement FIR filters using convolution
2. Implement IIR filters using difference equations
3. Apply fixed-point arithmetic for embedded DSP
4. Measure filter frequency response
5. Process real-time signals with the ADC/DAC

---

## Prerequisites

- Completion of Labs 5 and 6 (ADC/DAC)
- Understanding of filter theory (transfer functions, frequency response)
- Basic DSP concepts (convolution, difference equations)

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- PmodAD1 (ADC input)
- PmodDA2 (DAC output)
- Function generator
- Oscilloscope
- Spectrum analyzer (optional)

---

## Background

### FIR Filter Structure

An N-tap FIR filter computes:

$$y[n] = \sum_{k=0}^{N-1} h[k] \cdot x[n-k]$$

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FIR FILTER STRUCTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  x[n] ──┬───►[h₀]───┐                                                       │
│         │           ▼                                                        │
│         └──►[Z⁻¹]──►[h₁]──►(+)                                              │
│               │            ▲                                                 │
│               └──►[Z⁻¹]──►[h₂]──►(+)                                        │
│                     │            ▲                                           │
│                     └──►[Z⁻¹]──►[h₃]──►(+)───► y[n]                         │
│                           │            ▲                                     │
│                           └──────...───┘                                     │
│                                                                              │
│  Z⁻¹ = Unit delay                                                           │
│  hₖ  = Filter coefficients                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

![FIR Filter Diagram](../imgs/labs/lab7-img-001.png)

### IIR Filter Structure

A Direct Form I IIR filter:

$$y[n] = \sum_{k=0}^{M} b_k \cdot x[n-k] - \sum_{k=1}^{N} a_k \cdot y[n-k]$$

![IIR Filter Structure](../imgs/labs/lab7-img-003.png)

---

## Part 1: FIR Filter Implementation

### Low-Pass FIR Filter

```c
#include <stdint.h>
#include "xil_printf.h"

#define FIR_TAPS        31
#define FIXED_POINT_BITS 15

// Low-pass filter coefficients (31-tap, fc = 1kHz @ 10kHz sample rate)
// Scaled to Q15 fixed-point format
const int16_t firCoeffs[FIR_TAPS] = {
    -50,  -87,  -95,  -42,   85,  268,  456,  570,
    538,  320,  -72, -567, -1052, -1394, -1472, 22937,
    -1472, -1394, -1052, -567,  -72,  320,  538,  570,
    456,  268,   85,  -42,  -95,  -87,  -50
};

// Delay line (circular buffer)
int16_t firDelayLine[FIR_TAPS];
int firDelayIndex = 0;

void FIR_Init(void) {
    // Clear delay line
    for (int i = 0; i < FIR_TAPS; i++) {
        firDelayLine[i] = 0;
    }
    firDelayIndex = 0;
    xil_printf("FIR filter initialized (%d taps)\r\n", FIR_TAPS);
}

int16_t FIR_Process(int16_t input) {
    int32_t accumulator = 0;
    int index;
    
    // Insert new sample into delay line
    firDelayLine[firDelayIndex] = input;
    
    // Compute convolution
    index = firDelayIndex;
    for (int k = 0; k < FIR_TAPS; k++) {
        // Q15 × Q15 = Q30, then shift to Q15
        accumulator += (int32_t)firCoeffs[k] * (int32_t)firDelayLine[index];
        
        // Circular buffer decrement
        index--;
        if (index < 0) index = FIR_TAPS - 1;
    }
    
    // Update delay index
    firDelayIndex++;
    if (firDelayIndex >= FIR_TAPS) firDelayIndex = 0;
    
    // Scale result (Q30 to Q15)
    return (int16_t)(accumulator >> FIXED_POINT_BITS);
}
```

### High-Pass FIR Filter

```c
// High-pass filter (spectral inversion of low-pass)
const int16_t firCoeffsHP[FIR_TAPS] = {
    50,   87,   95,   42,  -85, -268, -456, -570,
    -538, -320,   72,  567, 1052, 1394, 1472, 9831,
    1472, 1394, 1052,  567,   72, -320, -538, -570,
    -456, -268,  -85,   42,   95,   87,   50
};
```

---

## Part 2: IIR Filter Implementation

### Second-Order IIR Section (Biquad)

```c
typedef struct {
    int16_t b0, b1, b2;  // Feedforward coefficients
    int16_t a1, a2;      // Feedback coefficients (a0 = 1)
    int32_t x1, x2;      // Input delay registers
    int32_t y1, y2;      // Output delay registers
} Biquad_t;

// Low-pass Butterworth, fc = 1kHz @ 10kHz, Q15 format
Biquad_t lpfBiquad = {
    .b0 = 1639,    // 0.05 in Q15
    .b1 = 3278,    // 0.10 in Q15
    .b2 = 1639,    // 0.05 in Q15
    .a1 = -21111,  // -0.6443 in Q15
    .a2 = 10998,   // 0.3357 in Q15
    .x1 = 0, .x2 = 0,
    .y1 = 0, .y2 = 0
};

void Biquad_Init(Biquad_t *bq) {
    bq->x1 = bq->x2 = 0;
    bq->y1 = bq->y2 = 0;
}

int16_t Biquad_Process(Biquad_t *bq, int16_t input) {
    int32_t x0 = (int32_t)input;
    int32_t y0;
    
    // Direct Form I
    // y[n] = b0*x[n] + b1*x[n-1] + b2*x[n-2] - a1*y[n-1] - a2*y[n-2]
    y0 = ((int32_t)bq->b0 * x0 +
          (int32_t)bq->b1 * bq->x1 +
          (int32_t)bq->b2 * bq->x2 -
          (int32_t)bq->a1 * bq->y1 -
          (int32_t)bq->a2 * bq->y2) >> FIXED_POINT_BITS;
    
    // Saturation to prevent overflow
    if (y0 > 32767) y0 = 32767;
    if (y0 < -32768) y0 = -32768;
    
    // Update delay registers
    bq->x2 = bq->x1;
    bq->x1 = x0;
    bq->y2 = bq->y1;
    bq->y1 = y0;
    
    return (int16_t)y0;
}
```

![Biquad Implementation](../imgs/labs/lab7-img-008.png)

---

## Part 3: Real-Time Signal Processing

### ADC → Filter → DAC Pipeline

```c
#include "FreeRTOS.h"
#include "task.h"

typedef enum {
    FILTER_BYPASS,
    FILTER_FIR_LP,
    FILTER_FIR_HP,
    FILTER_IIR_LP,
    FILTER_IIR_HP
} FilterMode_t;

volatile FilterMode_t activeFilter = FILTER_BYPASS;

void vFilterTask(void *pvParameters) {
    uint16_t adcValue;
    int16_t inputSample, outputSample;
    
    // Initialize hardware
    SPI_Init();
    DAC_Init();
    FIR_Init();
    Biquad_Init(&lpfBiquad);
    
    xil_printf("Real-time filter active\r\n");
    xil_printf("Keys: 0=bypass, 1=FIR-LP, 2=FIR-HP, 3=IIR-LP\r\n");
    
    for (;;) {
        // Read ADC (12-bit unsigned → 16-bit signed centered)
        adcValue = ADC_Read(0);
        inputSample = (int16_t)(adcValue - 2048) << 4;  // Center and scale
        
        // Apply selected filter
        switch (activeFilter) {
            case FILTER_BYPASS:
                outputSample = inputSample;
                break;
                
            case FILTER_FIR_LP:
                outputSample = FIR_Process(inputSample);
                break;
                
            case FILTER_FIR_HP:
                outputSample = FIR_ProcessHP(inputSample);  // HP version
                break;
                
            case FILTER_IIR_LP:
                outputSample = Biquad_Process(&lpfBiquad, inputSample);
                break;
                
            default:
                outputSample = inputSample;
        }
        
        // Write to DAC (convert back to 12-bit unsigned)
        uint16_t dacValue = (uint16_t)((outputSample >> 4) + 2048);
        dacValue &= 0x0FFF;
        DAC_Write(0, dacValue);
    }
}
```

![Signal Flow](../imgs/labs/lab7-img-012.png)

---

## Part 4: Filter Coefficient Design

### Using MATLAB/Octave

```matlab
% FIR Low-Pass Filter Design
fs = 10000;           % Sample rate (Hz)
fc = 1000;            % Cutoff frequency (Hz)
N = 30;               % Filter order (number of taps - 1)

% Design filter using window method
b = fir1(N, fc/(fs/2), 'low');

% Convert to Q15 fixed-point
b_q15 = round(b * 32768);

% Print C array
fprintf('const int16_t firCoeffs[%d] = {\n', length(b_q15));
for i = 1:length(b_q15)
    fprintf('    %d', b_q15(i));
    if i < length(b_q15)
        fprintf(',');
    end
    if mod(i, 8) == 0
        fprintf('\n');
    end
end
fprintf('};\n');

% Plot frequency response
freqz(b, 1, 1024, fs);
title('FIR Low-Pass Filter Response');
```

![MATLAB Filter Design](../imgs/labs/lab7-img-020.png)

### Pre-Computed Coefficients

| Filter Type | Cutoff | Taps | Coefficient Set |
|------------|--------|------|-----------------|
| Low-pass | 500 Hz | 31 | firCoeffs_LP500 |
| Low-pass | 1 kHz | 31 | firCoeffs_LP1k |
| Low-pass | 2 kHz | 31 | firCoeffs_LP2k |
| High-pass | 1 kHz | 31 | firCoeffs_HP1k |
| Band-pass | 500-1500 Hz | 51 | firCoeffs_BP |

---

## Part 5: Frequency Response Measurement

### Sweep Test

```c
#define SWEEP_POINTS    20
uint32_t testFrequencies[SWEEP_POINTS] = {
    50, 100, 200, 300, 500, 700, 1000, 1200, 1500,
    2000, 2500, 3000, 3500, 4000, 4500, 5000, 6000, 7000, 8000, 10000
};

void MeasureFrequencyResponse(void) {
    xil_printf("\r\n=== Frequency Response Measurement ===\r\n");
    xil_printf("Apply sine wave at each frequency and measure output amplitude\r\n\r\n");
    xil_printf("Freq (Hz) | Input Vpp | Output Vpp | Gain (dB)\r\n");
    xil_printf("----------|-----------|------------|----------\r\n");
    
    for (int i = 0; i < SWEEP_POINTS; i++) {
        xil_printf("%5lu     |           |            |          \r\n",
                   testFrequencies[i]);
    }
    
    xil_printf("\r\nUse function generator to apply test signals\r\n");
    xil_printf("Measure output with oscilloscope\r\n");
    xil_printf("Calculate: Gain_dB = 20 * log10(Vout / Vin)\r\n");
}
```

![Frequency Response](../imgs/labs/lab7-img-025.png)

---

## Part 6: Band-Pass Filter

### Cascaded Low-Pass and High-Pass

```c
// Band-pass: cascade HP (fc=500Hz) and LP (fc=2kHz)
int16_t BandPass_Process(int16_t input) {
    int16_t hpOutput, bpOutput;
    
    // First stage: High-pass at 500 Hz
    hpOutput = Biquad_Process(&hpfBiquad, input);
    
    // Second stage: Low-pass at 2 kHz
    bpOutput = Biquad_Process(&lpfBiquad2k, hpOutput);
    
    return bpOutput;
}
```

![Band-Pass Response](../imgs/labs/lab7-img-028.png)

---

## Part 7: Notch Filter

### Remove 60 Hz Hum

```c
// 60 Hz notch filter (Q = 30)
Biquad_t notch60Hz = {
    .b0 = 32735,   // ≈ 0.9992
    .b1 = -32668,  // ≈ -0.9971 × 2 × cos(2π×60/fs)
    .b2 = 32735,   // ≈ 0.9992
    .a1 = -32668,  // Same as b1
    .a2 = 32703,   // ≈ 0.9984
    .x1 = 0, .x2 = 0,
    .y1 = 0, .y2 = 0
};
```

---

## Deliverables

1. **Source Code**:
   - FIR filter implementation with multiple cutoff options
   - IIR biquad implementation
   - Real-time ADC → Filter → DAC pipeline
   - Filter selection user interface

2. **Measurements**:
   - Frequency response plots for each filter type
   - Screenshots showing input vs. filtered output
   - Timing measurements (processing latency)

3. **Lab Report**:
   - Filter design methodology
   - Frequency response comparison (measured vs. theoretical)
   - Fixed-point precision analysis
   - Real-time performance evaluation

---

## Test Procedure

1. **Basic Verification**:
   - Connect function generator to PmodAD1
   - Connect oscilloscope to PmodDA2 output
   - Apply 1 kHz sine wave
   - Verify bypass mode (input ≈ output)

2. **Filter Response**:
   - Enable low-pass filter
   - Sweep input frequency from 100 Hz to 5 kHz
   - Record output amplitude at each frequency
   - Verify cutoff frequency and rolloff

3. **Real-Time Performance**:
   - Measure processing loop timing
   - Calculate maximum achievable sample rate

---

## Performance Optimization

### Assembly-Optimized MAC

```c
// Multiply-accumulate using ARM DSP intrinsics
#ifdef __ARM_FEATURE_DSP
int32_t FIR_ProcessOptimized(int16_t input) {
    int32_t acc = 0;
    int16_t *pCoeff = (int16_t *)firCoeffs;
    int16_t *pDelay = &firDelayLine[firDelayIndex];
    
    // Use SMLAD for dual multiply-accumulate
    for (int i = 0; i < FIR_TAPS; i += 2) {
        int32_t coeffPair = *((int32_t *)pCoeff);
        int32_t delayPair = *((int32_t *)pDelay);
        acc = __SMLAD(coeffPair, delayPair, acc);
        pCoeff += 2;
        pDelay -= 2;
        if (pDelay < firDelayLine) pDelay = &firDelayLine[FIR_TAPS-2];
    }
    
    return acc >> FIXED_POINT_BITS;
}
#endif
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Output clipping | Check saturation logic, reduce input gain |
| Unstable IIR | Verify coefficient precision, check overflow |
| Frequency response wrong | Recalculate coefficients, check sample rate |
| Audible noise | Increase filter order, check ADC grounding |

---

## Reference Materials

- [Lab 7 Writeup](../pdfs/LAB%207%20Digital%20Filters.pdf)
- [Module 12: Digital Signal Processing](../modules/module12/)
- [Understanding Digital Signal Processing (Lyons)](https://www.analog.com/en/education/landing-pages/professor-teaching-resources/dsp-book-third-edition.html)
- [CMSIS-DSP Library](https://arm-software.github.io/CMSIS-DSP/latest/)
