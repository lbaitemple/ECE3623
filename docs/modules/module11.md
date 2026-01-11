---
title: "Module 11: Digital-to-Analog Conversion"
layout: default
parent: "Progression 4: Signal Processing"
grand_parent: "Course Modules"
nav_order: 2
---

# Module 11: Digital-to-Analog Conversion (DAC) and Waveform Generation
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Learning Objectives

By the end of this module, you will be able to:

1. Configure the PmodDA2 two-channel, 12-bit DAC
2. Generate various waveforms (sawtooth, triangular, sine, square)
3. Use GPIO switches to adjust amplitude and frequency
4. Build a complete function generator application
5. Understand reconstruction filtering concepts

---

## Overview

Digital-to-Analog Converters (DACs) enable embedded systems to produce analog outputs from digital values. Using the PmodDA2, students will configure a function generator capable of outputting various waveforms with adjustable parameters.

---

## DAC Fundamentals

### What is a DAC?

A DAC converts discrete digital codes into continuous analog voltages:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DIGITAL-TO-ANALOG CONVERSION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Digital Code      ┌────────────┐      Analog Voltage          │
│   (Binary)    ─────►│    DAC     │─────►  (Continuous)          │
│   0x800 (2048)      └────────────┘       1.65V (mid-scale)      │
│                                                                  │
│   Output Voltage:                                                │
│                        Code × V_ref                              │
│              V_out = ──────────────                             │
│                           2^n                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### DAC Output Equation

For an n-bit DAC:

$$V_{out} = \frac{Code \times V_{ref}}{2^n}$$

$$Code = \left\lfloor \frac{V_{out} \times 2^n}{V_{ref}} \right\rfloor$$

### Example Calculations

**12-bit DAC, Vref = 3.3V:**

| Desired Voltage | Code Calculation | Digital Code |
|-----------------|------------------|--------------|
| 0V | $0 \times 4096 / 3.3$ | 0x000 |
| 1.65V (mid) | $1.65 \times 4096 / 3.3$ | 0x800 (2048) |
| 3.3V (full) | $3.3 \times 4096 / 3.3$ | 0xFFF (4095) |

---

## PmodDA2 Specifications

The PmodDA2 contains two DAC121S101 12-bit DACs:

| Specification | Value |
|---------------|-------|
| Resolution | 12 bits |
| Channels | 2 (independent) |
| Output Range | 0 to VREF (3.3V) |
| Settling Time | 8 µs |
| Update Rate | Up to 1 MSPS |
| Interface | SPI |

### Data Format

The DAC121S101 uses a 16-bit SPI word:

```
┌────┬────┬────┬────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬────┬────┬────┬────┐
│ 15 │ 14 │ 13 │ 12 │ 11  │ 10  │  9  │  8  │  7  │  6  │  5  │  4  │ 3  │ 2  │ 1  │ 0  │
├────┴────┼────┴────┼─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴────┴────┴────┴────┤
│  PD1-0  │   X X   │                    12-bit DAC Data (D11-D0)                       │
│ (00=ON) │  (don't │                                                                   │
│         │  care)  │                                                                   │
└─────────┴─────────┴───────────────────────────────────────────────────────────────────┘

Power-Down Mode:
  00 = Normal operation
  01 = 1kΩ to GND
  10 = 100kΩ to GND  
  11 = High-Z
```

---

## DAC Driver Implementation

### Initialization

```c
#include "xspips.h"

XSpiPs DacSpiInstance;

int DAC_Init(void) {
    XSpiPs_Config *SpiConfig;
    int Status;
    
    SpiConfig = XSpiPs_LookupConfig(XPAR_XSPIPS_0_DEVICE_ID);
    if (SpiConfig == NULL) {
        return XST_FAILURE;
    }
    
    Status = XSpiPs_CfgInitialize(&DacSpiInstance, SpiConfig,
                                   SpiConfig->BaseAddress);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }
    
    // DAC121S101 uses SPI Mode 1 (CPOL=0, CPHA=1)
    XSpiPs_SetOptions(&DacSpiInstance, 
                      XSPIPS_MASTER_OPTION |
                      XSPIPS_CLK_PHASE_1_OPTION |
                      XSPIPS_FORCE_SSELECT_OPTION);
    
    XSpiPs_SetClkPrescaler(&DacSpiInstance, XSPIPS_CLK_PRESCALE_32);
    
    return XST_SUCCESS;
}
```

### Writing to DAC

```c
#define DAC_CHANNEL_A   0
#define DAC_CHANNEL_B   1
#define VREF            3.3f

void DAC_WriteCode(uint8_t channel, uint16_t code) {
    uint8_t txBuffer[2];
    uint8_t rxBuffer[2];
    
    // Mask to 12 bits, power-down bits = 00 (normal operation)
    code &= 0x0FFF;
    
    // Format: [PD1 PD0 X X D11-D8] [D7-D0]
    txBuffer[0] = (code >> 8) & 0x0F;    // Upper 4 bits
    txBuffer[1] = code & 0xFF;            // Lower 8 bits
    
    // Select channel (different chip select lines)
    XSpiPs_SetSlaveSelect(&DacSpiInstance, channel);
    
    // Send 16 bits
    XSpiPs_PolledTransfer(&DacSpiInstance, txBuffer, rxBuffer, 2);
}

void DAC_WriteVoltage(uint8_t channel, float voltage) {
    if (voltage < 0) voltage = 0;
    if (voltage > VREF) voltage = VREF;
    
    uint16_t code = (uint16_t)((voltage / VREF) * 4095.0f);
    DAC_WriteCode(channel, code);
}
```

---

## Waveform Generation

### Lookup Table Approach

Pre-computing waveform samples improves performance:

```c
#define TABLE_SIZE  256

uint16_t sineTable[TABLE_SIZE];
uint16_t sawtoothTable[TABLE_SIZE];
uint16_t triangleTable[TABLE_SIZE];

void InitWaveformTables(void) {
    for (int i = 0; i < TABLE_SIZE; i++) {
        float phase = (float)i / TABLE_SIZE;
        
        // Sine wave: 0 to 4095, centered at 2048
        float angle = 2.0f * 3.14159f * phase;
        sineTable[i] = (uint16_t)(2048 + 2047 * sinf(angle));
        
        // Sawtooth wave: 0 → 4095 linearly
        sawtoothTable[i] = (uint16_t)(phase * 4095);
        
        // Triangle wave: 0 → 4095 → 0
        if (i < TABLE_SIZE / 2) {
            triangleTable[i] = (uint16_t)((i * 4095) / (TABLE_SIZE / 2));
        } else {
            triangleTable[i] = (uint16_t)(4095 - ((i - TABLE_SIZE/2) * 4095) / (TABLE_SIZE / 2));
        }
    }
}
```

### Waveform Diagrams

```
SAWTOOTH:                    TRIANGLE:
    ▲                            ▲
4095│    /│   /│   /│        4095│   /\    /\    /\
    │   / │  / │  / │            │  /  \  /  \  /  \
    │  /  │ /  │ /  │            │ /    \/    \/    \
    │ /   │/   │/   │            │/                  \
   0└──────────────────►        0└──────────────────────►
         Time                          Time

SINE:                        SQUARE:
    ▲                            ▲
4095│  ╭──╮   ╭──╮           4095│ ┌──┐  ┌──┐  ┌──┐
    │ ╱    ╲ ╱    ╲              │ │  │  │  │  │  │
2048│╱      ╲      ╲             │ │  │  │  │  │  │
    │        ╲      ╱            │ │  │  │  │  │  │
   0│         ╰──╯  ╰──         0│─┘  └──┘  └──┘  └──
    └──────────────────►         └──────────────────────►
         Time                          Time
```

---

## Function Generator Application

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUNCTION GENERATOR SYSTEM                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │   GPIO   │───►│ Control  │───►│ Waveform │───►│   DAC    │ │
│   │ Switches │    │   Task   │    │   Task   │    │  Output  │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
│                         │                                        │
│                         ▼                                        │
│                   ┌──────────┐                                   │
│                   │ Settings │                                   │
│                   │• Waveform│                                   │
│                   │• Freq    │                                   │
│                   │• Amp     │                                   │
│                   └──────────┘                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Waveform Type Enumeration

```c
typedef enum {
    WAVE_SINE,
    WAVE_SQUARE,
    WAVE_TRIANGLE,
    WAVE_SAWTOOTH
} WaveformType_t;

typedef struct {
    WaveformType_t type;
    uint32_t frequency;     // Hz
    uint8_t amplitude;      // 0-100%
} WaveformSettings_t;

volatile WaveformSettings_t settings = {
    .type = WAVE_SINE,
    .frequency = 100,
    .amplitude = 100
};
```

### GPIO Switch Control

```c
#include "xgpio.h"

XGpio GpioSwitches;

void ReadSwitches(void) {
    uint8_t sw = XGpio_DiscreteRead(&GpioSwitches, 1);
    
    // SW[1:0] - Waveform selection
    settings.type = sw & 0x03;
    
    // SW[3:2] - Frequency selection
    switch ((sw >> 2) & 0x03) {
        case 0: settings.frequency = 100;  break;  // 100 Hz
        case 1: settings.frequency = 500;  break;  // 500 Hz
        case 2: settings.frequency = 1000; break;  // 1 kHz
        case 3: settings.frequency = 5000; break;  // 5 kHz
    }
    
    // SW[5:4] - Amplitude selection
    switch ((sw >> 4) & 0x03) {
        case 0: settings.amplitude = 25;  break;  // 25%
        case 1: settings.amplitude = 50;  break;  // 50%
        case 2: settings.amplitude = 75;  break;  // 75%
        case 3: settings.amplitude = 100; break;  // 100%
    }
}
```

### Waveform Generation Task

```c
void vWaveformTask(void *pvParameters) {
    uint32_t tableIndex = 0;
    uint16_t *currentTable;
    
    InitWaveformTables();
    DAC_Init();
    
    for (;;) {
        // Select waveform table
        switch (settings.type) {
            case WAVE_SINE:     currentTable = sineTable;     break;
            case WAVE_TRIANGLE: currentTable = triangleTable; break;
            case WAVE_SAWTOOTH: currentTable = sawtoothTable; break;
            case WAVE_SQUARE:   currentTable = NULL;          break;
        }
        
        uint16_t dacValue;
        
        if (settings.type == WAVE_SQUARE) {
            // Square wave: simple high/low
            dacValue = (tableIndex < TABLE_SIZE/2) ? 4095 : 0;
        } else {
            dacValue = currentTable[tableIndex];
        }
        
        // Apply amplitude scaling
        dacValue = (dacValue * settings.amplitude) / 100;
        
        // Write to DAC
        DAC_WriteCode(DAC_CHANNEL_A, dacValue);
        
        // Increment table index
        tableIndex = (tableIndex + 1) % TABLE_SIZE;
        
        // Calculate delay for desired frequency
        // Period = TABLE_SIZE * delay_per_sample
        // delay_per_sample = 1 / (frequency * TABLE_SIZE)
        uint32_t delay_us = 1000000 / (settings.frequency * TABLE_SIZE);
        usleep(delay_us);
    }
}
```

---

## Dual-Channel Operation

### Independent Channels

```c
void vDualChannelTask(void *pvParameters) {
    uint32_t indexA = 0;
    uint32_t indexB = 0;
    
    for (;;) {
        // Channel A: Sine wave at f
        DAC_WriteCode(DAC_CHANNEL_A, sineTable[indexA]);
        
        // Channel B: Sine wave at 2f (phase shifted for quadrature)
        DAC_WriteCode(DAC_CHANNEL_B, sineTable[(indexB * 2) % TABLE_SIZE]);
        
        indexA = (indexA + 1) % TABLE_SIZE;
        indexB = (indexB + 1) % TABLE_SIZE;
        
        usleep(100);  // Base timing
    }
}
```

---

## Reconstruction and Filtering

### The Staircase Effect

DAC output changes in discrete steps, producing harmonics:

```
Ideal Sine:     ╭────╮   ╭────╮
               ╱      ╲ ╱      ╲
              ╱        ╳        ╲
             ╱                   ╲
            ╱                     ╲

DAC Output:     ┌─┐ ┌─┐   ┌─┐
               ┌┘ │┌┘ └┐ ┌┘ └┐
              ┌┘  └┘   └┌┘   └┐
             ┌┘          │     └┐
            ─┘           │      └─
            (staircase approximation)
```

### Reconstruction Filter

A low-pass filter after the DAC smooths the output:

$$f_c \leq \frac{f_s}{2}$$

**Simple RC Filter Design:**

$$f_c = \frac{1}{2\pi RC}$$

**Example**: For fc = 10 kHz with C = 0.1 µF:
$$R = \frac{1}{2\pi \times 10000 \times 0.1 \times 10^{-6}} = 159 \Omega$$

---

## Lecture Materials

- [Week 11 Slides]({{ site.baseurl }}/docs/pdfs/Week%2011.pdf)
- [PmodAD1 and PmodDA2]({{ site.baseurl }}/docs/pdfs/PmodAD1%20and%20PmodDA2.pdf)
- [Waveform Generation]({{ site.baseurl }}/docs/pdfs/Waveform%20Generation.pdf)

---

## Reading Assignments

1. PmodDA2 Reference Manual
2. DAC121S101 Datasheet
3. Application notes on waveform generation

---

## Practice Questions

1. A 12-bit DAC has Vref = 3.3V. What code produces 2.0V output?

2. To generate a 1 kHz sine wave using a 256-point table, what is the required DAC update rate?

3. Design a function generator that can output frequencies from 100 Hz to 10 kHz. What is the minimum DAC update rate needed?

4. Explain why reconstruction filtering is necessary after a DAC.

5. How would you modify the amplitude of a sine wave from 100% to 50% using the lookup table approach?

6. Calculate the frequency achieved with TABLE_SIZE=256 and 10 µs per sample.

7. A sawtooth wave resets every 1 ms. What is its fundamental frequency?

---

## Summary

DACs enable digital systems to generate analog outputs for waveform generation, audio, and control applications. The PmodDA2 provides two independent 12-bit channels with SPI interface. By using lookup tables and adjustable parameters via GPIO switches, a complete function generator can be built. The techniques covered here—waveform synthesis, amplitude scaling, and frequency control—are foundational for signal generation in embedded systems.

---

## Next Module

[Module 12: Digital Signal Processing - FIR and IIR Filters →]({{ site.baseurl }}/docs/modules/module12.html)
| Update Rate | Up to 1 MSPS |
| Interface | SPI |
| Channels | 2 (DA2 has two DACs) |

### Pin Connections

| Pmod Pin | Signal | Zynq Connection |
|----------|--------|-----------------|
| 1 | ~SYNC (CS) | SPI SS |
| 2 | NC | - |
| 3 | D_IN (MOSI) | SPI MOSI |
| 4 | SCLK | SPI SCLK |
| 5 | GND | Ground |
| 6 | VCC | 3.3V |

### SPI Data Format

16-bit write format:
- Bits 15-14: Power-down mode (00 = normal operation)
- Bits 13-12: Don't care
- Bits 11-0: 12-bit DAC data

```
┌────┬────┬────┬────┬────────────────────────────┐
│ PD1│ PD0│ X  │ X  │  D11 D10 D9 ... D2 D1 D0   │
└────┴────┴────┴────┴────────────────────────────┘
  15   14   13   12   11                       0
```

---

## Implementation

### DAC Driver

```c
#include "xspips.h"

#define DAC_SPI_DEVICE_ID  XPAR_XSPIPS_0_DEVICE_ID

XSpiPs SpiInstance;

int DAC_Init(void)
{
    XSpiPs_Config *SpiConfig;
    int Status;
    
    SpiConfig = XSpiPs_LookupConfig(DAC_SPI_DEVICE_ID);
    Status = XSpiPs_CfgInitialize(&SpiInstance, SpiConfig,
                                   SpiConfig->BaseAddress);
    
    /* Master mode, CPOL=0, CPHA=1 (Mode 1 for DAC121S101) */
    XSpiPs_SetOptions(&SpiInstance, 
                      XSPIPS_MASTER_OPTION |
                      XSPIPS_CLK_PHASE_1_OPTION |
                      XSPIPS_FORCE_SSELECT_OPTION);
    
    /* Set clock prescaler */
    XSpiPs_SetClkPrescaler(&SpiInstance, XSPIPS_CLK_PRESCALE_32);
    
    return Status;
}

void DAC_Write(u16 value)
{
    u8 txBuf[2];
    u8 rxBuf[2];
    
    /* Format: 00XX DDDD DDDD DDDD */
    /* Normal operation (PD=00), 12-bit data */
    value &= 0x0FFF;  /* Mask to 12 bits */
    
    txBuf[0] = (value >> 8) & 0x0F;  /* Upper 4 bits */
    txBuf[1] = value & 0xFF;          /* Lower 8 bits */
    
    XSpiPs_SetSlaveSelect(&SpiInstance, 0x01);
    XSpiPs_PolledTransfer(&SpiInstance, txBuf, rxBuf, 2);
}

void DAC_SetVoltage(float voltage, float vref)
{
    if (voltage < 0) voltage = 0;
    if (voltage > vref) voltage = vref;
    
    u16 code = (u16)((voltage / vref) * 4095.0f);
    DAC_Write(code);
}
```

---

## Waveform Generation

### Lookup Table Approach

Pre-compute waveform values for efficiency.

```c
#define TABLE_SIZE  256

u16 sineTable[TABLE_SIZE];
u16 triangleTable[TABLE_SIZE];
u16 sawtoothTable[TABLE_SIZE];

void InitWaveformTables(void)
{
    for (int i = 0; i < TABLE_SIZE; i++) {
        /* Sine: 0 to 4095, centered at 2048 */
        float angle = (2.0f * M_PI * i) / TABLE_SIZE;
        sineTable[i] = (u16)(2048 + 2047 * sinf(angle));
        
        /* Triangle */
        if (i < TABLE_SIZE / 2) {
            triangleTable[i] = (u16)((i * 4095) / (TABLE_SIZE / 2));
        } else {
            triangleTable[i] = (u16)(4095 - ((i - TABLE_SIZE/2) * 4095) / (TABLE_SIZE / 2));
        }
        
        /* Sawtooth */
        sawtoothTable[i] = (u16)((i * 4095) / TABLE_SIZE);
    }
}
```

### Square Wave Generation

```c
void GenerateSquareWave(u32 frequency, u32 duration_ms)
{
    u32 half_period_us = 500000 / frequency;  /* microseconds */
    u32 cycles = (frequency * duration_ms) / 1000;
    
    for (u32 i = 0; i < cycles; i++) {
        DAC_Write(4095);  /* High */
        usleep(half_period_us);
        
        DAC_Write(0);     /* Low */
        usleep(half_period_us);
    }
}
```

### Continuous Waveform Task

```c
typedef enum {
    WAVE_SINE,
    WAVE_SQUARE,
    WAVE_TRIANGLE,
    WAVE_SAWTOOTH
} WaveformType;

volatile WaveformType currentWaveform = WAVE_SINE;
volatile u32 waveformFrequency = 100;  /* Hz */

void vWaveformTask(void *pvParameters)
{
    u32 tableIndex = 0;
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    /* Calculate delay for desired frequency */
    /* Period = TABLE_SIZE * delay */
    
    for(;;) {
        u16 value;
        
        switch (currentWaveform) {
            case WAVE_SINE:
                value = sineTable[tableIndex];
                break;
            case WAVE_SQUARE:
                value = (tableIndex < TABLE_SIZE/2) ? 4095 : 0;
                break;
            case WAVE_TRIANGLE:
                value = triangleTable[tableIndex];
                break;
            case WAVE_SAWTOOTH:
                value = sawtoothTable[tableIndex];
                break;
        }
        
        DAC_Write(value);
        
        tableIndex = (tableIndex + 1) % TABLE_SIZE;
        
        /* Adjust delay for frequency */
        u32 delay_us = 1000000 / (waveformFrequency * TABLE_SIZE);
        usleep(delay_us);
    }
}
```

---

## Function Generator Application

### System Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   UART      │────►│  Control    │────►│  Waveform   │
│  Commands   │     │    Task     │     │    Task     │
└─────────────┘     └─────────────┘     └─────────────┘
                           │                   │
                           ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐
                    │   Display   │     │    DAC      │
                    │    Task     │     │   Output    │
                    └─────────────┘     └─────────────┘
```

### Command Interface

```c
void ProcessCommand(char *cmd)
{
    if (strncmp(cmd, "WAVE ", 5) == 0) {
        if (strcmp(&cmd[5], "SINE") == 0) {
            currentWaveform = WAVE_SINE;
        } else if (strcmp(&cmd[5], "SQUARE") == 0) {
            currentWaveform = WAVE_SQUARE;
        } else if (strcmp(&cmd[5], "TRIANGLE") == 0) {
            currentWaveform = WAVE_TRIANGLE;
        } else if (strcmp(&cmd[5], "SAWTOOTH") == 0) {
            currentWaveform = WAVE_SAWTOOTH;
        }
    }
    else if (strncmp(cmd, "FREQ ", 5) == 0) {
        u32 freq = atoi(&cmd[5]);
        if (freq >= 1 && freq <= 10000) {
            waveformFrequency = freq;
        }
    }
    else if (strncmp(cmd, "AMP ", 4) == 0) {
        /* Set amplitude (0-100%) */
        u32 amp = atoi(&cmd[4]);
        waveformAmplitude = (amp > 100) ? 100 : amp;
    }
}
```

---

## Output Reconstruction

### The Staircase Effect

DAC output changes in discrete steps, creating a "staircase" waveform:

```
Ideal:    ╱╲╱╲╱╲╱╲
Actual:   ┌┐┌┐┌┐┌┐
          └┘└┘└┘└┘
```

### Reconstruction Filter

A low-pass filter smooths the output:

$$f_c < \frac{f_s}{2}$$

```
DAC Output ──►[Low-Pass Filter]──► Smooth Analog
                    │
              Cutoff = fs/2
```

### Simple RC Filter

$$f_c = \frac{1}{2\pi RC}$$

For $f_c = 10 kHz$ with $C = 0.1 \mu F$:

$$R = \frac{1}{2\pi \cdot 10000 \cdot 0.1 \times 10^{-6}} = 159 \Omega$$

---

## Lab Connection

### Lab 6: PmodDA2 Function Generator

In this lab, you will:
1. Configure SPI for PmodDA2 interface
2. Generate sine, square, triangle waveforms
3. Implement frequency and amplitude control
4. Build a complete function generator

**Lab Materials**: [Lab 6 - Function Generator]({{ site.baseurl }}/docs/pdfs/LAB%206%20PmodDA2%20-%20Function%20Generator.pdf)

---

## Lecture Materials

- [PmodAD1 and PmodDA2]({{ site.baseurl }}/docs/pdfs/PmodAD1%20and%20PmodDA2.pdf)
- [Zynq SPI Peripherals ADC & DAC]({{ site.baseurl }}/docs/pdfs/Zynq%20SPI%20Peripherals%20ADC%20%26%20DAC.pdf)

---

## Reading Assignments

1. PmodDA2 Reference Manual
2. DAC121S101 Datasheet
3. Application notes on waveform generation

---

## Practice Questions

1. A 10-bit DAC has $V_{ref} = 5V$. What output voltage does code 512 produce?

2. You need to generate a 1 kHz sine wave using a 256-point lookup table. What is the required DAC update rate?

3. Explain why a reconstruction filter is needed after a DAC.

4. Design an RC low-pass filter with cutoff at 5 kHz using a 0.01 μF capacitor.

5. How would you modify the sine table to change the amplitude to 50%?

---

## Summary

DACs enable digital systems to produce analog outputs for waveform generation, audio, and control applications. We covered DAC fundamentals, the PmodDA2 interface, waveform generation techniques, and reconstruction filtering. Combined with ADC knowledge, you can now build complete analog I/O systems. The next module covers digital signal processing.

---

## Next Module

[Module 12: Digital Filtering →]({{ site.baseurl }}/docs/modules/module12.html)
