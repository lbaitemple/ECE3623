---
title: "Module 12: Digital Signal Processing"
layout: default
parent: "Progression 4: Signal Processing"
grand_parent: Course Modules
nav_order: 3
---

# Module 12: Digital Signal Processing - FIR and IIR Filters
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Design and implement FIR (Finite Impulse Response) filters
2. Design and implement IIR (Infinite Impulse Response) filters
3. Calculate filter zeros and poles
4. Convert between difference equations and transfer functions
5. Locate zeros and poles on the z-plane
6. Implement real-time filters in embedded C

---

## Overview

Digital filters are essential for signal processing in embedded systems. This module covers the theory and implementation of both FIR and IIR filters, including their mathematical representation and practical coding techniques.

---

## Filter Fundamentals

### Why Digital Filters?

| Analog Filters | Digital Filters |
|----------------|-----------------|
| R, L, C components | Software algorithms |
| Component tolerances | Exact coefficients |
| Fixed after construction | Programmable |
| No latency | Computational delay |
| Simple filters only | Complex responses possible |

### Filter Types by Response

```
LOW-PASS:              HIGH-PASS:             BAND-PASS:
│Gain                  │Gain                  │Gain
│████                  │     ████             │   ███
│████                  │     ████             │   ███
│████─────             │─────████             │───███───
└────────►freq         └────────►freq         └────────►freq
    fc                      fc                  f1   f2
```

---

## FIR Filters (Finite Impulse Response)

### Difference Equation

The FIR filter output is a weighted sum of current and past inputs:

$$y[n] = \sum_{k=0}^{N-1} b_k \cdot x[n-k]$$

$$y[n] = b_0 x[n] + b_1 x[n-1] + b_2 x[n-2] + \cdots + b_{N-1} x[n-N+1]$$

Where:
- $y[n]$ = output at sample n
- $x[n]$ = input at sample n
- $b_k$ = filter coefficients (taps)
- $N$ = filter order + 1 (number of taps)

### Transfer Function

The z-domain transfer function for FIR:

$$H(z) = \sum_{k=0}^{N-1} b_k z^{-k} = b_0 + b_1 z^{-1} + b_2 z^{-2} + \cdots + b_{N-1} z^{-(N-1)}$$

### FIR Block Diagram

```
x[n]──┬─────►[b₀]────────────────────────────────────┐
      │                                              │
      ▼                                              ▼
    [z⁻¹]──┬──►[b₁]─────────────────────────────────(+)
           │                                         │
           ▼                                         ▼
         [z⁻¹]──┬──►[b₂]────────────────────────────(+)
                │                                    │
                ▼                                    ▼
              [z⁻¹]──┬──►[b₃]──────────────────────(+)──► y[n]
                     │                              │
                    ...                            ...
```

### FIR Zeros

FIR filters have **zeros only** (no poles except at origin). The zeros are roots of:

$$H(z) = b_0 + b_1 z^{-1} + \cdots = 0$$

**Example**: 3-tap FIR with coefficients [1, -2, 1]:

$$H(z) = 1 - 2z^{-1} + z^{-2} = \frac{z^2 - 2z + 1}{z^2} = \frac{(z-1)^2}{z^2}$$

**Zeros**: z = 1 (double zero)
**Poles**: z = 0 (double pole at origin, always present in FIR)

---

## FIR Implementation

```c
#define FIR_TAPS    17

// Filter coefficients (low-pass example)
float fir_coeffs[FIR_TAPS] = {
    0.003, 0.008, 0.020, 0.040, 0.065,
    0.090, 0.110, 0.120, 0.120, 0.110,
    0.090, 0.065, 0.040, 0.020, 0.008,
    0.003, 0.001
};

// Delay line (circular buffer)
float fir_buffer[FIR_TAPS] = {0};
int fir_index = 0;

float FIR_Filter(float input) {
    float output = 0;
    int i, buf_idx;
    
    // Store new sample in circular buffer
    fir_buffer[fir_index] = input;
    
    // Compute convolution (dot product)
    buf_idx = fir_index;
    for (i = 0; i < FIR_TAPS; i++) {
        output += fir_coeffs[i] * fir_buffer[buf_idx];
        buf_idx--;
        if (buf_idx < 0) buf_idx = FIR_TAPS - 1;
    }
    
    // Update circular buffer index
    fir_index++;
    if (fir_index >= FIR_TAPS) fir_index = 0;
    
    return output;
}
```

---

## IIR Filters (Infinite Impulse Response)

### Difference Equation

IIR filters use both input samples AND past output samples (feedback):

$$y[n] = \sum_{k=0}^{M} b_k \cdot x[n-k] - \sum_{k=1}^{N} a_k \cdot y[n-k]$$

Expanded form:

$$y[n] = b_0 x[n] + b_1 x[n-1] + \cdots - a_1 y[n-1] - a_2 y[n-2] - \cdots$$

### Transfer Function

$$H(z) = \frac{B(z)}{A(z)} = \frac{b_0 + b_1 z^{-1} + \cdots + b_M z^{-M}}{1 + a_1 z^{-1} + \cdots + a_N z^{-N}}$$

### IIR Block Diagram (Direct Form I)

```
                    Feedforward (FIR part)
x[n]──┬─────►[b₀]──────────────────────────────────┐
      │                                            │
      ▼                                            ▼
    [z⁻¹]──┬──►[b₁]───────────────────────────────(+)
           │                                       │
           ▼                                       ▼
         [z⁻¹]──►[b₂]─────────────────────────────(+)──► y[n]
                                                   ▲
    [z⁻¹]◄──┬──◄[-a₁]──────────────────────────────┤
           │                                        │
           ▼                                        │
         [z⁻¹]◄──[-a₂]──────────────────────────────┘
                    Feedback (IIR part)
```

---

## Zeros and Poles

### Definitions

- **Zeros**: Values of z where H(z) = 0 (numerator = 0)
- **Poles**: Values of z where H(z) → ∞ (denominator = 0)

### Stability Criterion

**An IIR filter is stable if and only if all poles lie INSIDE the unit circle** (|z| < 1)

### Z-Plane Visualization

```
              Im
               ▲
               │    Unit Circle
         ×     │     ╱
          ╲    │   ╱      ○ = zero
           ╲   │  ╱       × = pole
            ╲  │ ╱
    ─────────╲─│╱────────────► Re
              ╲│╱
               │
         ×     │
               │
               
   STABLE: All poles (×) inside circle
   UNSTABLE: Any pole outside circle
```

### Example: Second-Order IIR (Biquad)

$$H(z) = \frac{b_0 + b_1 z^{-1} + b_2 z^{-2}}{1 + a_1 z^{-1} + a_2 z^{-2}}$$

**Coefficients** (Butterworth low-pass, fc = 0.1fs):
- b = [0.0675, 0.1349, 0.0675]
- a = [1.0, -1.1430, 0.4128]

**Finding poles** (roots of denominator):
$$1 - 1.143z^{-1} + 0.4128z^{-2} = 0$$
$$z^2 - 1.143z + 0.4128 = 0$$

Using quadratic formula:
$$z = \frac{1.143 \pm \sqrt{1.306 - 1.651}}{2} = 0.572 \pm j0.412$$

Pole magnitude: $|z| = \sqrt{0.572^2 + 0.412^2} = 0.705 < 1$ ✓ **STABLE**

---

## IIR Implementation (Biquad)

```c
typedef struct {
    float b[3];     // Numerator coefficients [b0, b1, b2]
    float a[3];     // Denominator coefficients [1, a1, a2]
    float x[3];     // Input history [x[n], x[n-1], x[n-2]]
    float y[3];     // Output history [y[n], y[n-1], y[n-2]]
} Biquad_t;

// Initialize biquad section
void Biquad_Init(Biquad_t *bq, float *b, float *a) {
    for (int i = 0; i < 3; i++) {
        bq->b[i] = b[i];
        bq->a[i] = a[i];
        bq->x[i] = 0;
        bq->y[i] = 0;
    }
}

// Process one sample
float Biquad_Process(Biquad_t *bq, float input) {
    // Shift input history
    bq->x[2] = bq->x[1];
    bq->x[1] = bq->x[0];
    bq->x[0] = input;
    
    // Shift output history
    bq->y[2] = bq->y[1];
    bq->y[1] = bq->y[0];
    
    // Compute output: y[n] = b0*x[n] + b1*x[n-1] + b2*x[n-2]
    //                       - a1*y[n-1] - a2*y[n-2]
    bq->y[0] = bq->b[0] * bq->x[0] +
               bq->b[1] * bq->x[1] +
               bq->b[2] * bq->x[2] -
               bq->a[1] * bq->y[1] -
               bq->a[2] * bq->y[2];
    
    return bq->y[0];
}
```

### Cascaded Biquads for Higher Orders

```c
#define NUM_SECTIONS  2  // 4th order = 2 biquad sections

Biquad_t filter_sections[NUM_SECTIONS];

float Filter_4thOrder(float input) {
    float output = input;
    
    // Cascade: output of section k is input to section k+1
    for (int k = 0; k < NUM_SECTIONS; k++) {
        output = Biquad_Process(&filter_sections[k], output);
    }
    
    return output;
}
```

---

## Converting Between Representations

### Difference Equation → Transfer Function

Given: $y[n] = 0.5x[n] + 0.5x[n-1] + 0.8y[n-1]$

1. Take z-transform: $Y(z) = 0.5X(z) + 0.5z^{-1}X(z) + 0.8z^{-1}Y(z)$
2. Rearrange: $Y(z) - 0.8z^{-1}Y(z) = 0.5X(z) + 0.5z^{-1}X(z)$
3. Factor: $Y(z)(1 - 0.8z^{-1}) = X(z)(0.5 + 0.5z^{-1})$
4. Result: $H(z) = \frac{Y(z)}{X(z)} = \frac{0.5 + 0.5z^{-1}}{1 - 0.8z^{-1}}$

### Transfer Function → Difference Equation

Given: $H(z) = \frac{1 + 2z^{-1} + z^{-2}}{1 - 0.5z^{-1} + 0.25z^{-2}}$

1. Cross-multiply: $Y(z)(1 - 0.5z^{-1} + 0.25z^{-2}) = X(z)(1 + 2z^{-1} + z^{-2})$
2. Inverse z-transform:
   $y[n] - 0.5y[n-1] + 0.25y[n-2] = x[n] + 2x[n-1] + x[n-2]$
3. Solve for y[n]:
   $y[n] = x[n] + 2x[n-1] + x[n-2] + 0.5y[n-1] - 0.25y[n-2]$

---

## FIR vs IIR Comparison

| Property | FIR | IIR |
|----------|-----|-----|
| **Stability** | Always stable | Can be unstable |
| **Phase** | Linear phase possible | Non-linear phase |
| **Order for sharp cutoff** | High (many taps) | Low (few coefficients) |
| **Computation** | More multiplies | Fewer multiplies |
| **Delay** | Higher (N/2 samples) | Lower |
| **Design** | Easier (windowing) | More complex |

---

## Real-Time Filtering Example

```c
#include "FreeRTOS.h"
#include "task.h"

Biquad_t lowpass_filter;

void vFilterTask(void *pvParameters) {
    float b[] = {0.0675, 0.1349, 0.0675};
    float a[] = {1.0, -1.1430, 0.4128};
    
    Biquad_Init(&lowpass_filter, b, a);
    
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    for (;;) {
        // Read ADC
        uint16_t raw = ADC_Read(0);
        float input = ADC_ToVoltage(raw);
        
        // Apply IIR filter
        float filtered = Biquad_Process(&lowpass_filter, input);
        
        // Output to DAC
        DAC_WriteVoltage(0, filtered);
        
        // Maintain 1 kHz sample rate
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(1));
    }
}
```

---

## Lecture Materials

- [Week 12 Slides - Digital Filtering]({{ site.baseurl }}/ece3623/Week%2012%20Filtering.pdf)
- [FIR and IIR Filter Design]({{ site.baseurl }}/ece3623/Filter%20Design.pdf)
- [Z-Transform Review]({{ site.baseurl }}/ece3623/Z-Transform.pdf)

---

## Reading Assignments

1. Introduction to Digital Signal Processing, Chapter on Digital Filters
2. Understanding Digital Signal Processing (Lyons), Chapters 5-6
3. Fixed-point filter implementation techniques

---

## Practice Questions

1. Write the difference equation for an FIR filter with coefficients b = [1, 2, 1].

2. Given $H(z) = \frac{1 - z^{-2}}{1 - 0.9z^{-1}}$, find the zeros and poles.

3. Is the filter with pole at z = 0.95 stable? What about z = 1.05?

4. Convert this difference equation to a transfer function:
   $y[n] = x[n] - x[n-2] + 0.5y[n-1]$

5. A 4th-order Butterworth filter is implemented as cascaded biquads. How many biquad sections are needed?

6. What is the main advantage of FIR filters in audio applications?

7. Plot the approximate location of poles for an IIR filter with a = [1, -1.5, 0.7].

---

## Summary

FIR filters use weighted sums of input samples and are always stable with possible linear phase. IIR filters include feedback from past outputs, achieving sharper responses with fewer coefficients but requiring stability analysis. Understanding the relationship between difference equations, transfer functions, and pole-zero locations is essential for filter design and analysis. The z-plane provides a visual tool for understanding filter behavior and stability.

---

## Next Module

[Module 13: High-Level Synthesis (HLS) Design Flow →](../module13/)
      ...         ...
       │           │
      [z⁻¹]──►[bN-1]┘
```

### FIR Implementation

```c
#define FIR_ORDER  16
#define FIR_TAPS   (FIR_ORDER + 1)

/* Filter coefficients (example: low-pass) */
float firCoeffs[FIR_TAPS] = {
    0.0039, 0.0078, 0.0195, 0.0391, 0.0625,
    0.0859, 0.1055, 0.1172, 0.1172, 0.1055,
    0.0859, 0.0625, 0.0391, 0.0195, 0.0078,
    0.0039, 0.0020
};

/* Delay line (circular buffer) */
float firBuffer[FIR_TAPS] = {0};
int firIndex = 0;

float FIR_Filter(float input)
{
    float output = 0;
    
    /* Store new sample */
    firBuffer[firIndex] = input;
    
    /* Compute convolution */
    int bufIdx = firIndex;
    for (int i = 0; i < FIR_TAPS; i++) {
        output += firCoeffs[i] * firBuffer[bufIdx];
        bufIdx--;
        if (bufIdx < 0) bufIdx = FIR_TAPS - 1;
    }
    
    /* Update index */
    firIndex++;
    if (firIndex >= FIR_TAPS) firIndex = 0;
    
    return output;
}
```

---

## IIR Filters (Infinite Impulse Response)

### Characteristics
- Output depends on inputs AND past outputs (feedback)
- Can be unstable if poorly designed
- Sharper cutoff with fewer coefficients
- Non-linear phase (typically)

### Difference Equation

$$y[n] = \sum_{k=0}^{M} b_k \cdot x[n-k] - \sum_{k=1}^{N} a_k \cdot y[n-k]$$

### Direct Form II Implementation

```c
#define IIR_ORDER  2  /* Second-order (biquad) */

/* Coefficients for 2nd order low-pass Butterworth */
float b[3] = {0.0675, 0.1349, 0.0675};  /* Numerator */
float a[3] = {1.0, -1.1430, 0.4128};    /* Denominator */

/* State variables */
float x_hist[3] = {0, 0, 0};  /* Input history */
float y_hist[3] = {0, 0, 0};  /* Output history */

float IIR_Filter(float input)
{
    /* Shift history */
    x_hist[2] = x_hist[1];
    x_hist[1] = x_hist[0];
    x_hist[0] = input;
    
    y_hist[2] = y_hist[1];
    y_hist[1] = y_hist[0];
    
    /* Compute output */
    float output = b[0] * x_hist[0] + 
                   b[1] * x_hist[1] + 
                   b[2] * x_hist[2] -
                   a[1] * y_hist[1] - 
                   a[2] * y_hist[2];
    
    y_hist[0] = output;
    
    return output;
}
```

### Biquad Cascade

For higher-order filters, cascade second-order sections:

```c
typedef struct {
    float b[3];  /* Numerator coefficients */
    float a[3];  /* Denominator coefficients */
    float x[3];  /* Input history */
    float y[3];  /* Output history */
} BiquadSection;

float Biquad_Process(BiquadSection *bq, float input)
{
    bq->x[2] = bq->x[1];
    bq->x[1] = bq->x[0];
    bq->x[0] = input;
    
    bq->y[2] = bq->y[1];
    bq->y[1] = bq->y[0];
    
    bq->y[0] = bq->b[0] * bq->x[0] + 
               bq->b[1] * bq->x[1] + 
               bq->b[2] * bq->x[2] -
               bq->a[1] * bq->y[1] - 
               bq->a[2] * bq->y[2];
    
    return bq->y[0];
}

/* 4th order filter = 2 biquad sections */
BiquadSection filter[2];

float Filter4thOrder(float input)
{
    float output = Biquad_Process(&filter[0], input);
    output = Biquad_Process(&filter[1], output);
    return output;
}
```

---

## Fixed-Point Arithmetic

### Why Fixed-Point?

- Faster than floating-point on many embedded processors
- No FPU required
- Deterministic timing
- Lower power consumption

### Q Format Notation

**Qm.n format**: m integer bits, n fractional bits

Example: Q15 (Q0.15) uses 16 bits with 15 fractional bits.

Range: $-1.0$ to $+0.999969...$

### Conversion

```c
/* Float to Q15 */
#define FLOAT_TO_Q15(x)  ((int16_t)((x) * 32768.0f))

/* Q15 to float */
#define Q15_TO_FLOAT(x)  ((float)(x) / 32768.0f)

/* Q15 multiplication */
int16_t Q15_Mult(int16_t a, int16_t b)
{
    int32_t result = (int32_t)a * (int32_t)b;
    return (int16_t)(result >> 15);
}
```

### Fixed-Point FIR Filter

```c
#define FIR_TAPS  17

/* Q15 coefficients */
int16_t firCoeffs_Q15[FIR_TAPS] = {
    128, 256, 640, 1280, 2048,
    2816, 3456, 3840, 3840, 3456,
    2816, 2048, 1280, 640, 256,
    128, 64
};

int16_t firBuffer_Q15[FIR_TAPS] = {0};
int firIndex = 0;

int16_t FIR_Filter_Q15(int16_t input)
{
    int32_t accumulator = 0;
    
    firBuffer_Q15[firIndex] = input;
    
    int bufIdx = firIndex;
    for (int i = 0; i < FIR_TAPS; i++) {
        accumulator += (int32_t)firCoeffs_Q15[i] * 
                       (int32_t)firBuffer_Q15[bufIdx];
        bufIdx--;
        if (bufIdx < 0) bufIdx = FIR_TAPS - 1;
    }
    
    firIndex = (firIndex + 1) % FIR_TAPS;
    
    /* Scale back to Q15 */
    return (int16_t)(accumulator >> 15);
}
```

---

## Moving Average Filter

Simple but effective noise reduction:

$$y[n] = \frac{1}{N} \sum_{k=0}^{N-1} x[n-k]$$

```c
#define MA_LENGTH  8

int16_t maBuffer[MA_LENGTH] = {0};
int maIndex = 0;
int32_t maSum = 0;

int16_t MovingAverage(int16_t input)
{
    /* Subtract oldest, add newest */
    maSum -= maBuffer[maIndex];
    maSum += input;
    maBuffer[maIndex] = input;
    
    maIndex = (maIndex + 1) % MA_LENGTH;
    
    return (int16_t)(maSum / MA_LENGTH);
}
```

---

## Real-Time Filtering Example

```c
void vFilterTask(void *pvParameters)
{
    u16 adcRaw;
    float adcVoltage;
    float filtered;
    
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(1);  /* 1 kHz */
    
    for(;;) {
        /* Read ADC */
        adcRaw = ADC_Read();
        adcVoltage = ADC_ToVoltage(adcRaw, 3.3f);
        
        /* Apply filter */
        filtered = FIR_Filter(adcVoltage);
        
        /* Output to DAC */
        DAC_SetVoltage(filtered, 3.3f);
        
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
```

---

## Lab Connection

### Lab 7: Digital Filters

In this lab, you will:
1. Implement a moving average filter
2. Design and implement an FIR low-pass filter
3. Compare filtered vs unfiltered signals
4. Use fixed-point arithmetic for efficiency

**Lab Materials**: [Lab 7 - Digital Filters]({{ site.baseurl }}/ece3623/LAB%207%20Digital%20Filters.pdf)

---

## Lecture Materials

- [Week 9 - Filtering]({{ site.baseurl }}/ece3623/Week%209-T%20Filtering.pdf)
- [Fixed Point Arithmetic Supplement]({{ site.baseurl }}/ece3623/Fixed%20Point%20Arithmetic%20-%20Supplement-6.pdf)

---

## Reading Assignments

1. Introduction to digital filter design
2. Fixed-point arithmetic for DSP
3. FIR vs IIR filter comparison

---

## Practice Questions

1. What is the main advantage of FIR filters over IIR filters?

2. A 16-tap FIR filter is sampled at 10 kHz. What is the filter delay in milliseconds?

3. Convert -0.5 to Q15 format.

4. Design a 4-point moving average filter difference equation.

5. Why might an IIR filter become unstable? How is this related to the denominator coefficients?

---

## Summary

Digital filters are essential tools for signal processing in embedded systems. FIR filters offer stability and linear phase, while IIR filters provide efficient sharp cutoffs. Fixed-point arithmetic enables efficient real-time implementation on resource-constrained processors. The next module explores hardware acceleration using High-Level Synthesis.

---

## Next Module

[Module 13: High-Level Synthesis →](../module13/)
