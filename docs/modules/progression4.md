---
title: "Progression 4: Signal Processing"
layout: default
nav_order: 4
parent: Course Modules
has_children: true
---

# Progression 4: Signal Processing

Understand analog-to-digital and digital-to-analog conversion, and implement digital filters.

---

## Overview

This progression covers the complete signal processing chain:

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [Module 10](module10/) | Analog-to-Digital Conversion | Sampling, quantization, Nyquist theorem, ADC resolution |
| [Module 11](module11/) | Digital-to-Analog Conversion | DAC output, waveform generation, function generator |
| [Module 12](module12/) | Digital Signal Processing | FIR filters, IIR filters, z-transform, poles and zeros |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                SIGNAL PROCESSING PROGRESSION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Module 10             Module 11             Module 12          │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐       │
│  │   ADC    │          │   DAC    │          │  Digital │       │
│  │  Theory  │─────────►│ Waveform │─────────►│  Filters │       │
│  │          │          │Generation│          │ FIR/IIR  │       │
│  └──────────┘          └──────────┘          └──────────┘       │
│                                                                  │
│   Analog → Digital      Digital → Analog     Digital Processing  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    COMPLETE SIGNAL CHAIN                  │   │
│  │  Sensor ──► ADC ──► DSP Filter ──► DAC ──► Output        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Completion of Progression 3 (Timing & Peripherals)
- Basic understanding of signals and frequency concepts

---

## Key Equations

**Nyquist Sampling Theorem:**
$$f_s \geq 2 \cdot f_{max}$$

**ADC Resolution:**
$$LSB = \frac{V_{ref}}{2^n}$$

**FIR Filter:**
$$y[n] = \sum_{k=0}^{N-1} b_k \cdot x[n-k]$$

**IIR Filter:**
$$H(z) = \frac{B(z)}{A(z)} = \frac{b_0 + b_1 z^{-1} + \cdots}{1 + a_1 z^{-1} + \cdots}$$

---

## Outcomes

After completing this progression, you will be able to:

1. Apply Nyquist theorem and calculate ADC parameters
2. Generate analog waveforms using DAC modules
3. Design and implement FIR and IIR digital filters
4. Analyze filter stability using poles and zeros
