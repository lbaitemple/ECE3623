---
title: "Lab 5: PmodAD1 ADC Interface"
layout: default
parent: Laboratory Exercises
nav_order: 5
---

# Lab 5: PmodAD1 ADC Interface
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In this lab, you will interface the PmodAD1 12-bit analog-to-digital converter with the Zybo Z7 using SPI communication. You will read analog voltages, convert them to digital values, and display the results through UART.

**Related Modules**: [Module 9: ADC and SPI Communication]({{ site.baseurl }}/docs/modules/module09/), [Module 10: Analog-to-Digital Conversion]({{ site.baseurl }}/docs/modules/module10/)

---

## Learning Objectives

By completing this lab, you will be able to:

1. Configure the Zynq PS SPI controller
2. Understand SPI timing and clock modes
3. Interface with the AD7476 12-bit ADC
4. Convert digital codes to voltage readings
5. Implement multi-channel data acquisition

---

## Prerequisites

- Completion of Labs 1-4
- Understanding of SPI protocol basics
- Familiarity with ADC concepts (sampling, quantization)

---

## Required Hardware

- Zybo Z7-10 or Z7-20 development board
- **PmodAD1** - Dual 12-bit ADC module
- Jumper wires for analog input
- (Optional) Function generator for test signals

---

## Background

### PmodAD1 Specifications

| Parameter | Value |
|-----------|-------|
| Resolution | 12 bits |
| Channels | 2 (independent) |
| Max Sample Rate | 1 MSPS |
| Input Range | 0 to VREF (3.3V) |
| Interface | SPI (Mode 0 or 3) |

### SPI Communication

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPI BUS SIGNALS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│      ┌────────────┐                    ┌────────────┐           │
│      │            │◄──── MISO ─────────│            │           │
│      │   Zynq     │                    │  PmodAD1   │           │
│      │   (SPI     │───── SCLK ────────►│  (AD7476)  │           │
│      │   Master)  │───── CS ──────────►│            │           │
│      │            │                    │            │           │
│      └────────────┘                    └────────────┘           │
│                                                                  │
│  Note: MOSI not used - ADC is read-only                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

![PmodAD1 Connection]({{ site.baseurl }}/docs/imgs/pdf_images/pmod/img-000.png)

### AD7476 Data Frame

```
16-bit SPI transfer:
┌────┬────┬────┬────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬────┬────┬────┬────┐
│ 15 │ 14 │ 13 │ 12 │ 11  │ 10  │  9  │  8  │  7  │  6  │  5  │  4  │ 3  │ 2  │ 1  │ 0  │
├────┴────┴────┴────┼─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴────┴────┴────┴────┤
│   Leading Zeros   │                    12-bit ADC Result                              │
│      (0000)       │                         D11 ... D0                                │
└───────────────────┴───────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Hardware Setup

### Vivado Block Design

1. Create new Vivado project for Zybo Z7
2. Add ZYNQ7 Processing System
3. Configure **SPI 0** with EMIO (for Pmod connection)
4. Add GPIO for chip select control

![Vivado SPI Configuration]({{ site.baseurl }}/docs/imgs/pdf_images/pmod/img-002.png)

### Pmod Connection

Connect PmodAD1 to Pmod JA:

| PmodAD1 Pin | Zybo JA Pin | Signal |
|-------------|-------------|--------|
| Pin 1 (CS1) | JA1 | Chip Select Ch1 |
| Pin 2 (D0) | JA2 | MISO (Data Ch1) |
| Pin 3 (D1) | JA3 | MISO (Data Ch2) |
| Pin 4 (SCLK) | JA4 | SPI Clock |
| Pin 5 (GND) | GND | Ground |
| Pin 6 (VCC) | VCC | 3.3V Power |

---

## Part 2: SPI Driver

### SPI Initialization

```c
#include "xspips.h"
#include "xil_printf.h"

#define SPI_DEVICE_ID    XPAR_XSPIPS_0_DEVICE_ID
#define VREF             3.3f

XSpiPs SpiInstance;

int SPI_Init(void) {
    XSpiPs_Config *SpiConfig;
    int Status;
    
    // Look up SPI configuration
    SpiConfig = XSpiPs_LookupConfig(SPI_DEVICE_ID);
    if (SpiConfig == NULL) {
        xil_printf("SPI Config lookup failed\r\n");
        return XST_FAILURE;
    }
    
    // Initialize SPI driver
    Status = XSpiPs_CfgInitialize(&SpiInstance, SpiConfig,
                                   SpiConfig->BaseAddress);
    if (Status != XST_SUCCESS) {
        xil_printf("SPI Init failed\r\n");
        return XST_FAILURE;
    }
    
    // Perform self-test
    Status = XSpiPs_SelfTest(&SpiInstance);
    if (Status != XST_SUCCESS) {
        xil_printf("SPI Self-test failed\r\n");
        return XST_FAILURE;
    }
    
    // Configure: Master, Mode 0, manual CS
    XSpiPs_SetOptions(&SpiInstance, 
                      XSPIPS_MASTER_OPTION |
                      XSPIPS_FORCE_SSELECT_OPTION);
    
    // Set clock prescaler (~1 MHz SPI clock)
    XSpiPs_SetClkPrescaler(&SpiInstance, XSPIPS_CLK_PRESCALE_64);
    
    xil_printf("SPI Initialized successfully\r\n");
    return XST_SUCCESS;
}
```

### ADC Read Function

```c
uint16_t ADC_Read(uint8_t channel) {
    uint8_t txBuffer[2] = {0x00, 0x00};  // Dummy bytes for clock
    uint8_t rxBuffer[2] = {0x00, 0x00};
    
    // Select chip select line for channel (0 or 1)
    XSpiPs_SetSlaveSelect(&SpiInstance, channel);
    
    // Perform 16-bit SPI transfer
    XSpiPs_PolledTransfer(&SpiInstance, txBuffer, rxBuffer, 2);
    
    // Extract 12-bit value from received data
    // Bits 15-12 are zeros, bits 11-0 are the ADC value
    uint16_t adcValue = ((rxBuffer[0] & 0x0F) << 8) | rxBuffer[1];
    
    return adcValue;
}

float ADC_ToVoltage(uint16_t code) {
    return ((float)code / 4095.0f) * VREF;
}
```

---

## Part 3: Basic ADC Reading

### Single Channel Reading

```c
void vADCTask(void *pvParameters) {
    uint16_t adcCode;
    float voltage;
    
    if (SPI_Init() != XST_SUCCESS) {
        xil_printf("ADC initialization failed!\r\n");
        vTaskDelete(NULL);
    }
    
    xil_printf("Starting ADC readings...\r\n");
    xil_printf("Channel 0 connected to analog input\r\n\r\n");
    
    for (;;) {
        // Read from channel 0
        adcCode = ADC_Read(0);
        voltage = ADC_ToVoltage(adcCode);
        
        // Print results (integer math for xil_printf)
        int voltageInt = (int)voltage;
        int voltageFrac = (int)((voltage - voltageInt) * 1000);
        
        xil_printf("ADC Code: %4d  Voltage: %d.%03d V\r\n",
                   adcCode, voltageInt, voltageFrac);
        
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

### Expected Output

```
Starting ADC readings...
Channel 0 connected to analog input

ADC Code: 2048  Voltage: 1.650 V
ADC Code: 2050  Voltage: 1.652 V
ADC Code: 2046  Voltage: 1.648 V
ADC Code: 4095  Voltage: 3.300 V   (max input)
ADC Code:    0  Voltage: 0.000 V   (min input)
```

---

## Part 4: Dual Channel Reading

### Reading Both Channels

```c
typedef struct {
    uint16_t ch0_code;
    uint16_t ch1_code;
    float ch0_voltage;
    float ch1_voltage;
} DualADCReading_t;

void vDualADCTask(void *pvParameters) {
    DualADCReading_t reading;
    
    SPI_Init();
    
    for (;;) {
        // Read both channels
        reading.ch0_code = ADC_Read(0);
        reading.ch1_code = ADC_Read(1);
        reading.ch0_voltage = ADC_ToVoltage(reading.ch0_code);
        reading.ch1_voltage = ADC_ToVoltage(reading.ch1_code);
        
        xil_printf("CH0: %4d (%d.%03dV)  CH1: %4d (%d.%03dV)\r\n",
                   reading.ch0_code,
                   (int)reading.ch0_voltage,
                   (int)((reading.ch0_voltage - (int)reading.ch0_voltage) * 1000),
                   reading.ch1_code,
                   (int)reading.ch1_voltage,
                   (int)((reading.ch1_voltage - (int)reading.ch1_voltage) * 1000));
        
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}
```

---

## Part 5: Continuous Sampling

### High-Speed Data Acquisition

```c
#define SAMPLE_BUFFER_SIZE  256

uint16_t sampleBuffer[SAMPLE_BUFFER_SIZE];

void vSamplingTask(void *pvParameters) {
    int sampleIndex = 0;
    uint32_t startTime, endTime;
    
    SPI_Init();
    
    xil_printf("Acquiring %d samples...\r\n", SAMPLE_BUFFER_SIZE);
    
    // Capture samples as fast as possible
    startTime = xTaskGetTickCount();
    
    for (sampleIndex = 0; sampleIndex < SAMPLE_BUFFER_SIZE; sampleIndex++) {
        sampleBuffer[sampleIndex] = ADC_Read(0);
    }
    
    endTime = xTaskGetTickCount();
    
    // Calculate sample rate
    uint32_t elapsedMs = (endTime - startTime) * portTICK_PERIOD_MS;
    float sampleRate = (float)SAMPLE_BUFFER_SIZE / ((float)elapsedMs / 1000.0f);
    
    xil_printf("Acquisition complete!\r\n");
    xil_printf("Time: %lu ms\r\n", elapsedMs);
    xil_printf("Sample rate: %d samples/sec\r\n", (int)sampleRate);
    
    // Print first 10 samples
    xil_printf("\r\nFirst 10 samples:\r\n");
    for (int i = 0; i < 10; i++) {
        float v = ADC_ToVoltage(sampleBuffer[i]);
        xil_printf("  [%d]: %4d = %d.%03dV\r\n", i, sampleBuffer[i],
                   (int)v, (int)((v - (int)v) * 1000));
    }
    
    vTaskDelete(NULL);
}
```

---

## Part 6: Statistics and Analysis

### Calculate Min, Max, Average

```c
void AnalyzeSamples(uint16_t *buffer, int count) {
    uint32_t sum = 0;
    uint16_t minVal = 4095;
    uint16_t maxVal = 0;
    
    for (int i = 0; i < count; i++) {
        sum += buffer[i];
        if (buffer[i] < minVal) minVal = buffer[i];
        if (buffer[i] > maxVal) maxVal = buffer[i];
    }
    
    float average = (float)sum / count;
    
    xil_printf("\r\n=== Sample Analysis ===\r\n");
    xil_printf("Count: %d samples\r\n", count);
    xil_printf("Min: %d (%d.%03dV)\r\n", minVal,
               (int)ADC_ToVoltage(minVal),
               (int)((ADC_ToVoltage(minVal) - (int)ADC_ToVoltage(minVal)) * 1000));
    xil_printf("Max: %d (%d.%03dV)\r\n", maxVal,
               (int)ADC_ToVoltage(maxVal),
               (int)((ADC_ToVoltage(maxVal) - (int)ADC_ToVoltage(maxVal)) * 1000));
    xil_printf("Avg: %d.%02d\r\n", (int)average, 
               (int)((average - (int)average) * 100));
    xil_printf("Peak-Peak: %d codes\r\n", maxVal - minVal);
}
```

---

## Deliverables

1. **Source Code**:
   - SPI initialization and ADC read functions
   - Single and dual channel reading
   - Continuous sampling implementation

2. **Screenshots**:
   - Vivado block design with SPI
   - Serial output showing ADC readings
   - Voltage measurements with known inputs

3. **Lab Report**:
   - Accuracy analysis (compare to multimeter)
   - Sample rate measurements
   - Noise analysis (standard deviation)

---

## Test Procedure

1. Connect PmodAD1 to Zybo JA port
2. Connect CH0 input to GND → Should read ~0
3. Connect CH0 input to 3.3V → Should read ~4095
4. Use potentiometer or voltage divider for variable input
5. (Optional) Apply sine wave and capture samples

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| All zeros read | Check Pmod connection, verify CS toggling |
| Values stuck at 4095 | Check for floating input, verify VREF |
| Noisy readings | Add decoupling capacitor, average samples |
| SPI timeout | Check clock prescaler, verify connections |

---

## Reference Materials

- [Lab 5 Writeup](../pdfs/LAB%205%20PmodAD1%20Vivado%20SDK%20Project(2023_Fall).docx)
- [PmodAD1 and PmodDA2 Guide](../pdfs/PmodAD1%20and%20PmodDA2.pdf)
- [Zynq SPI Peripherals](../pdfs/Zynq%20SPI%20Peripherals%20ADC%20%26%20DAC.pdf)
- [Module 9: ADC and SPI Communication]({{ site.baseurl }}/docs/modules/module09/)
- [Module 10: Analog-to-Digital Conversion]({{ site.baseurl }}/docs/modules/module10/)
