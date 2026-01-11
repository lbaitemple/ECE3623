---
title: "Module 10: Analog-to-Digital Conversion"
layout: default
parent: "Progression 4: Signal Processing"
grand_parent: "Course Modules"
nav_order: 1
---

# Module 10: Analog-to-Digital Conversion (ADC) and SPI
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the sampling and quantization process
2. Apply the Nyquist sampling theorem correctly
3. Calculate ADC resolution, LSB size, and quantization error
4. Interface the PmodAD1 (AD7476) 12-bit ADC
5. Implement ADC reading in embedded C code

---

## Overview

Analog-to-Digital Converters (ADCs) are the bridge between the continuous analog world and discrete digital processing. This module explores the theory behind sampling and quantization, and demonstrates practical ADC implementation using the PmodAD1 module.

---

## The ADC Process

### Two-Step Conversion

Converting an analog signal to digital involves two distinct processes:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANALOG-TO-DIGITAL CONVERSION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Continuous      ┌──────────┐      Discrete     ┌──────────┐    │
│  Analog    ─────►│ SAMPLING │─────► Time   ────►│QUANTIZING│───►│
│  Signal          └──────────┘      Samples      └──────────┘    │
│                       │                              │           │
│              Time discretization          Amplitude discretization│
│                       ▼                              ▼           │
│               x(t) → x[n]                    x[n] → Digital Code │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Sampling

**Sampling** converts a continuous-time signal into a discrete-time sequence by taking measurements at regular intervals.

$$x[n] = x(nT_s)$$

Where:
- $x[n]$ = sampled value at index n
- $T_s$ = sampling period
- $f_s = 1/T_s$ = sampling frequency

### Quantization

**Quantization** maps continuous amplitude values to a finite set of discrete levels.

```
Amplitude
    ▲
4.0V├─────────────────────────────────────────── Level 7 (111)
    │               ●
3.5V├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Level 6 (110)
    │           ●       ●
3.0V├─────────────────────────────────────────── Level 5 (101)
    │       ●               ●
2.5V├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Level 4 (100)
    │   ●                       ●
2.0V├─────────────────────────────────────────── Level 3 (011)
    │                               ●
1.5V├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Level 2 (010)
    │●                                  ●
1.0V├─────────────────────────────────────────── Level 1 (001)
    │                                       ●
0.5V├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Level 0 (000)
    └──┬───┬───┬───┬───┬───┬───┬───┬───┬───┬──► Time
       0   1   2   3   4   5   6   7   8   9
                     Sample number
```

---

## Nyquist Sampling Theorem

### The Fundamental Rule

To accurately represent a signal, the sampling frequency must be **at least twice** the highest frequency component in the signal:

$$f_s \geq 2 \cdot f_{max}$$

- $f_s$ = sampling frequency (samples per second)
- $f_{max}$ = highest frequency in the signal (bandwidth)
- $f_N = f_s/2$ = Nyquist frequency

### Example Calculation

**Problem**: A signal contains frequencies from 0 to 5 kHz. What is the minimum sampling rate?

$$f_s \geq 2 \times 5\text{ kHz} = 10 \text{ kHz}$$

**Practical rule**: Use $f_s \geq 2.5 \times f_{max}$ to allow for filter roll-off.

---

## Aliasing

### What is Aliasing?

When $f_s < 2 \cdot f_{max}$, high-frequency components **fold back** and appear as lower frequencies, corrupting the signal:

```
┌─────────────────────────────────────────────────────────────────┐
│                         ALIASING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Original 10 kHz signal:    ∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿                  │
│                                                                  │
│  Sampled at 8 kHz:          •   •   •   •   •   •   •           │
│                             (under-sampled!)                     │
│                                                                  │
│  Reconstructed signal:      ∿  ∿  ∿  ∿  ∿  ∿  ∿                 │
│  (appears as 2 kHz!)        ↑ ALIAS!                            │
│                                                                  │
│  Alias frequency: |f_signal - k × f_s| where k makes result      │
│                   fall within 0 to f_s/2                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Alias Frequency Formula

$$f_{alias} = |f_{signal} - k \cdot f_s|$$

Where k is chosen so $0 \leq f_{alias} \leq f_s/2$

**Example**: 10 kHz signal sampled at 8 kHz:
$$f_{alias} = |10 - 1 \times 8| = 2 \text{ kHz}$$

### Anti-Aliasing Filter

A low-pass filter **before** the ADC removes frequencies above the Nyquist limit:

```
Analog     ┌─────────────┐     ┌─────────────┐     Digital
Input ────►│Anti-Aliasing│────►│     ADC     │────► Output
           │   Filter    │     │             │
           └─────────────┘     └─────────────┘
                 │
         Cutoff ≤ fs/2
```

---

## ADC Resolution and Quantization

### Resolution

An n-bit ADC divides the input range into $2^n$ discrete levels.

| Resolution | Levels | LSB (3.3V ref) |
|------------|--------|----------------|
| 8-bit | 256 | 12.9 mV |
| 10-bit | 1024 | 3.22 mV |
| **12-bit** | **4096** | **0.806 mV** |
| 14-bit | 16384 | 0.201 mV |
| 16-bit | 65536 | 50.4 µV |

### LSB Calculation

The Least Significant Bit (LSB) is the smallest voltage change detectable:

$$LSB = \frac{V_{ref}}{2^n}$$

**For PmodAD1 (12-bit, 3.3V reference):**
$$LSB = \frac{3.3V}{4096} = 0.806 \text{ mV}$$

### Quantization Error

The maximum error introduced by quantization is:

$$\text{Quantization Error} = \pm \frac{LSB}{2} = \pm \frac{V_{ref}}{2^{n+1}}$$

**For 12-bit ADC**: Error = ±0.403 mV

---

## Digital Code Conversion

### Voltage to Digital Code

$$\text{Digital Code} = \left\lfloor \frac{V_{in} \times 2^n}{V_{ref}} \right\rfloor$$

### Digital Code to Voltage

$$V_{in} = \frac{\text{Digital Code} \times V_{ref}}{2^n}$$

### Example

**12-bit ADC, Vref = 3.3V, Vin = 1.65V:**

$$\text{Code} = \left\lfloor \frac{1.65 \times 4096}{3.3} \right\rfloor = \left\lfloor 2048 \right\rfloor = 2048$$

**Verification:**
$$V_{in} = \frac{2048 \times 3.3}{4096} = 1.65 \text{ V}$$

---

## PmodAD1 (AD7476) Details

### AD7476A 12-bit ADC

| Specification | Value |
|---------------|-------|
| Resolution | 12 bits |
| Channels | 2 (independent ADCs) |
| Sampling Rate | Up to 1 MSPS |
| Input Range | 0 to VREF (3.3V) |
| Interface | SPI (Mode 0 or 3) |
| Power | 3.3V, low power |

### Data Frame Format

The AD7476 outputs 16 bits per conversion:

```
┌────┬────┬────┬────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬────┬────┬────┬────┐
│ 15 │ 14 │ 13 │ 12 │ 11  │ 10  │  9  │  8  │  7  │  6  │  5  │  4  │ 3  │ 2  │ 1  │ 0  │
├────┴────┴────┴────┼─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴────┴────┴────┴────┤
│   Leading Zeros   │                    12-bit ADC Result (MSB first)                  │
│      (0000)       │                         D11 ... D0                                │
└───────────────────┴───────────────────────────────────────────────────────────────────┘
```

---

## ADC Implementation Code

### Complete ADC Driver

```c
#include "xspips.h"
#include "xil_printf.h"

#define ADC_SPI_DEVICE_ID    XPAR_XSPIPS_0_DEVICE_ID
#define VREF                 3.3f

XSpiPs SpiInstance;

int ADC_Init(void) {
    XSpiPs_Config *SpiConfig;
    int Status;
    
    // Look up SPI configuration
    SpiConfig = XSpiPs_LookupConfig(ADC_SPI_DEVICE_ID);
    if (SpiConfig == NULL) {
        return XST_FAILURE;
    }
    
    // Initialize SPI driver
    Status = XSpiPs_CfgInitialize(&SpiInstance, SpiConfig,
                                   SpiConfig->BaseAddress);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }
    
    // Configure: Master mode, Mode 0 (CPOL=0, CPHA=0)
    XSpiPs_SetOptions(&SpiInstance, 
                      XSPIPS_MASTER_OPTION |
                      XSPIPS_FORCE_SSELECT_OPTION);
    
    // Set clock prescaler for ~1 MHz SPI clock
    XSpiPs_SetClkPrescaler(&SpiInstance, XSPIPS_CLK_PRESCALE_64);
    
    return XST_SUCCESS;
}

uint16_t ADC_Read(uint8_t channel) {
    uint8_t txBuffer[2] = {0x00, 0x00};  // Dummy data to generate clocks
    uint8_t rxBuffer[2] = {0x00, 0x00};
    
    // Select chip select line for channel
    XSpiPs_SetSlaveSelect(&SpiInstance, channel);
    
    // Perform 16-bit SPI transfer
    XSpiPs_PolledTransfer(&SpiInstance, txBuffer, rxBuffer, 2);
    
    // Extract 12-bit value: bits 11-0 of received 16-bit word
    uint16_t adcValue = ((rxBuffer[0] & 0x0F) << 8) | rxBuffer[1];
    
    return adcValue;
}

float ADC_ToVoltage(uint16_t code) {
    return ((float)code / 4095.0f) * VREF;
}

float ADC_ReadVoltage(uint8_t channel) {
    uint16_t code = ADC_Read(channel);
    return ADC_ToVoltage(code);
}
```

### FreeRTOS Sampling Task

```c
#include "FreeRTOS.h"
#include "task.h"

#define SAMPLE_RATE_HZ    1000
#define SAMPLE_PERIOD_MS  (1000 / SAMPLE_RATE_HZ)

void vADCSampleTask(void *pvParameters) {
    uint16_t adcCode;
    float voltage;
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    // Initialize ADC
    if (ADC_Init() != XST_SUCCESS) {
        xil_printf("ADC Init Failed!\r\n");
        vTaskDelete(NULL);
    }
    
    for (;;) {
        // Read 12-bit value
        adcCode = ADC_Read(0);
        
        // Convert to voltage
        voltage = ADC_ToVoltage(adcCode);
        
        // Print result
        xil_printf("Code: %4d  Voltage: %d.%03d V\r\n",
                   adcCode,
                   (int)voltage,
                   (int)((voltage - (int)voltage) * 1000));
        
        // Maintain precise sample timing
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SAMPLE_PERIOD_MS));
    }
}
```

---

## Signal Conditioning

### Input Protection

```
                    ┌─────────────────┐
Sensor   ┌────┐    │                 │
Output ──┤ R  ├────┤    ADC Input    │
         └────┘    │                 │
            │      └─────────────────┘
            │
           ═╪═ C (optional filter)
            │
          ──┴── GND
```

### Voltage Divider for Higher Voltages

For input voltages > VREF, use a resistor divider:

$$V_{ADC} = V_{in} \times \frac{R_2}{R_1 + R_2}$$

**Example**: To measure 0-10V with 3.3V ADC:
$$\frac{R_2}{R_1 + R_2} = \frac{3.3}{10} = 0.33$$

Use R1 = 20kΩ, R2 = 10kΩ

---

## Lecture Materials

- [Week 10 Slides]({{ site.baseurl }}/docs/pdfs/Week%2010.pdf)
- [PmodAD1 and PmodDA2]({{ site.baseurl }}/docs/pdfs/PmodAD1%20and%20PmodDA2.pdf)
- [Zynq SPI Peripherals ADC & DAC]({{ site.baseurl }}/docs/pdfs/Zynq%20SPI%20Peripherals%20ADC%20%26%20DAC.pdf)

---

## Reading Assignments

1. PmodAD1 Reference Manual
2. AD7476A Datasheet (timing and interface sections)
3. *Embedded Systems Design*, Chapter on Data Acquisition

---

## Practice Questions

1. A 10-bit ADC has Vref = 5V. Calculate the LSB size and maximum quantization error.

2. You need to sample a 4 kHz audio signal. What is the minimum sampling rate? What would you use in practice?

3. A signal at 15 kHz is sampled at 12 kHz. What alias frequency appears?

4. The PmodAD1 returns code 3000. With Vref = 3.3V, what is the input voltage?

5. Design an anti-aliasing filter specification for a 10 kHz sampling rate system.

6. Calculate the number of samples needed to capture exactly 10 cycles of a 60 Hz signal at 1 kHz sampling rate.

7. Explain why the AD7476 requires 16 clock cycles to output a 12-bit result.

---

## Summary

ADCs convert analog signals to digital through sampling and quantization. The Nyquist theorem sets the minimum sampling rate to avoid aliasing. Resolution determines the smallest detectable voltage change. The PmodAD1's AD7476 provides 12-bit resolution with SPI interface, suitable for many embedded applications. Proper implementation requires understanding of timing, data extraction, and voltage conversion.

---

## Next Module

[Module 11: Digital-to-Analog Conversion (DAC) →]({{ site.baseurl }}/docs/modules/module11.html)
Original signal (10 kHz):  ∿∿∿∿∿∿∿∿
Sampled at 8 kHz:          • • • • • • • •
Reconstructed (2 kHz!):    ∿  ∿  ∿  ∿
```

### Anti-Aliasing Filter

A low-pass filter before the ADC removes frequencies above $f_s/2$.

```
Input ──►[Anti-Alias LPF]──►[ADC]──► Digital
                │
         Cutoff < fs/2
```

---

## ADC Types

| Type | Speed | Resolution | Applications |
|------|-------|------------|--------------|
| **Flash** | Very fast | 6-8 bits | Video, radar |
| **SAR** | Medium-fast | 12-18 bits | Data acquisition |
| **Delta-Sigma** | Slow | 16-24 bits | Audio, sensors |
| **Pipeline** | Fast | 10-16 bits | Communications |

The PmodAD1 uses an **AD7476A SAR ADC**.

---

## PmodAD1 Interface

### AD7476A Specifications

| Parameter | Value |
|-----------|-------|
| Resolution | 12 bits |
| Sampling Rate | Up to 1 MSPS |
| Input Range | 0 to $V_{ref}$ |
| Interface | SPI |
| Channels | 2 (AD1 has two AD7476) |

### Pin Connections

| Pmod Pin | Signal | Zynq Connection |
|----------|--------|-----------------|
| 1 | ~CS | SPI SS |
| 2 | NC | - |
| 3 | MISO | SPI MISO |
| 4 | SCLK | SPI SCLK |
| 5 | GND | Ground |
| 6 | VCC | 3.3V |

### SPI Timing

The AD7476 returns 16 bits per conversion:
- Bits 15-12: Leading zeros
- Bits 11-0: 12-bit conversion result

```
CS    ─┐                                    ┌─
       └────────────────────────────────────┘

SCLK  ────┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬────
          │1│2│3│4│5│6│7│8│9│...│14│15│16│

MISO  ────<0><0><0><0><D11><D10>...<D1><D0>──
            Leading zeros    12-bit Data
```

---

## Implementation

### Hardware Setup in Vivado

1. Create block design with Zynq PS
2. Enable SPI peripheral (SPI0 or SPI1)
3. Connect SPI pins to Pmod connector
4. Generate bitstream

### Software Driver

```c
#include "xspips.h"

#define ADC_SPI_DEVICE_ID  XPAR_XSPIPS_0_DEVICE_ID
#define ADC_BYTES          2

XSpiPs SpiInstance;

int ADC_Init(void)
{
    XSpiPs_Config *SpiConfig;
    int Status;
    
    SpiConfig = XSpiPs_LookupConfig(ADC_SPI_DEVICE_ID);
    Status = XSpiPs_CfgInitialize(&SpiInstance, SpiConfig,
                                   SpiConfig->BaseAddress);
    
    /* Master mode, CPOL=0, CPHA=0 (Mode 0) */
    XSpiPs_SetOptions(&SpiInstance, 
                      XSPIPS_MASTER_OPTION |
                      XSPIPS_FORCE_SSELECT_OPTION);
    
    /* Set clock: ~1 MHz */
    XSpiPs_SetClkPrescaler(&SpiInstance, XSPIPS_CLK_PRESCALE_64);
    
    return Status;
}

u16 ADC_Read(void)
{
    u8 txBuf[ADC_BYTES] = {0xFF, 0xFF};
    u8 rxBuf[ADC_BYTES] = {0, 0};
    
    XSpiPs_SetSlaveSelect(&SpiInstance, 0x01);
    XSpiPs_PolledTransfer(&SpiInstance, txBuf, rxBuf, ADC_BYTES);
    
    /* Extract 12-bit value from received data */
    u16 adcValue = ((rxBuf[0] & 0x0F) << 8) | rxBuf[1];
    
    return adcValue;
}

float ADC_ToVoltage(u16 code, float vref)
{
    return ((float)code / 4095.0f) * vref;
}
```

### Continuous Sampling

```c
#define NUM_SAMPLES  1000
#define SAMPLE_PERIOD_MS  1  /* 1 kHz sampling */

u16 samples[NUM_SAMPLES];

void SampleTask(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();
    u32 sampleIndex = 0;
    
    for(;;) {
        /* Read ADC */
        samples[sampleIndex] = ADC_Read();
        
        sampleIndex++;
        if (sampleIndex >= NUM_SAMPLES) {
            sampleIndex = 0;
            /* Process complete buffer */
            ProcessSamples(samples, NUM_SAMPLES);
        }
        
        /* Maintain precise timing */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SAMPLE_PERIOD_MS));
    }
}
```

---

## Signal Processing Basics

### DC Offset Removal

```c
float RemoveDCOffset(u16 *samples, int n)
{
    /* Calculate mean */
    u32 sum = 0;
    for (int i = 0; i < n; i++) {
        sum += samples[i];
    }
    float mean = (float)sum / n;
    
    return mean;
}

void ApplyDCRemoval(float *output, u16 *input, int n, float offset)
{
    for (int i = 0; i < n; i++) {
        output[i] = (float)input[i] - offset;
    }
}
```

### Finding Min/Max

```c
void FindMinMax(u16 *samples, int n, u16 *min, u16 *max)
{
    *min = samples[0];
    *max = samples[0];
    
    for (int i = 1; i < n; i++) {
        if (samples[i] < *min) *min = samples[i];
        if (samples[i] > *max) *max = samples[i];
    }
}

float CalculatePeakToPeak(u16 *samples, int n, float vref)
{
    u16 min, max;
    FindMinMax(samples, n, &min, &max);
    return ADC_ToVoltage(max - min, vref);
}
```

### RMS Calculation

$$V_{RMS} = \sqrt{\frac{1}{N}\sum_{i=0}^{N-1} V_i^2}$$

```c
float CalculateRMS(float *samples, int n)
{
    float sumSquares = 0;
    for (int i = 0; i < n; i++) {
        sumSquares += samples[i] * samples[i];
    }
    return sqrtf(sumSquares / n);
}
```

---

## Lab Connection

### Lab 5: PmodAD1 Vivado SDK Project

In this lab, you will:
1. Configure SPI for PmodAD1 interface
2. Read ADC values and convert to voltage
3. Sample a signal at fixed rate
4. Display results via UART

**Lab Materials**: [Lab 5 - PmodAD1 Project]({{ site.baseurl }}/docs/pdfs/LAB%205%20PmodAD1%20Vivado%20SDK%20Project(2023_Fall).pdf)

---

## Lecture Materials

- [PmodAD1 and PmodDA2]({{ site.baseurl }}/docs/pdfs/PmodAD1%20and%20PmodDA2.pdf)
- [Zynq SPI Peripherals ADC & DAC]({{ site.baseurl }}/docs/pdfs/Zynq%20SPI%20Peripherals%20ADC%20%26%20DAC.pdf)

---

## Reading Assignments

1. PmodAD1 Reference Manual
2. AD7476 Datasheet (sections on timing and interface)
3. Application notes on ADC selection and interfacing

---

## Practice Questions

1. A 10-bit ADC has $V_{ref} = 5V$. What is the LSB size?

2. You need to sample a 5 kHz signal. What is the minimum sampling rate?

3. An ADC returns code 2048 with 12 bits and $V_{ref} = 3.3V$. What is the input voltage?

4. Why is an anti-aliasing filter needed before an ADC?

5. Calculate the number of samples needed to capture 5 complete cycles of a 100 Hz signal at 10 kHz sampling rate.

---

## Summary

ADCs convert analog signals to digital values for processing. Key concepts include resolution, sampling rate, and the Nyquist theorem. The PmodAD1 provides a practical SPI-based ADC interface for the Zynq platform. These skills enable data acquisition from real-world sensors. The next module covers DACs for signal generation.

---

## Next Module

[Module 11: Digital-to-Analog Conversion →]({{ site.baseurl }}/docs/modules/module11.html)
