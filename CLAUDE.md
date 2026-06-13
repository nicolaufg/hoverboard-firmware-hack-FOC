# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Field Oriented Control (FOC) firmware for stock hoverboard mainboards, targeting the STM32F103RCT6 (ARM Cortex-M3, 72 MHz, 256KB Flash, 48KB RAM). It replaces the original hoverboard firmware with advanced motor control supporting commutation, sinusoidal, and FOC control types.

## Build Commands

### Using Make (ARM GCC toolchain required: `arm-none-eabi-gcc`)

```bash
make                            # Build ELF, HEX, and BIN
make clean                      # Remove build artifacts
make format                     # Format Src/ and Inc/ with clang-format
make flash                      # Flash via ST-Link (st-flash)
make unlock                     # Unlock STM32 via OpenOCD
make -e VARIANT=VARIANT_ADC     # Build a specific variant
```

Outputs go to `build/` as `hover.elf`, `hover.hex`, and `hover.bin`.

### Using PlatformIO

```bash
pio run                         # Build active variant (set in platformio.ini)
```

Set the active variant by uncommenting the desired `default_envs` line in `platformio.ini`.

### No test suite exists. CI (`.github/workflows/build_on_commit.yml`) builds `VARIANT_ADC` with Make and runs `pio run`.

## Variant Selection

The firmware compiles one input variant at a time. Set via:
- `make -e VARIANT=<name>` (Makefile)
- `default_envs` in `platformio.ini`
- `#define <VARIANT_NAME>` in `Inc/config.h` (Keil/manual)

Available variants: `VARIANT_ADC`, `VARIANT_USART`, `VARIANT_NUNCHUK`, `VARIANT_PPM`, `VARIANT_PWM`, `VARIANT_IBUS`, `VARIANT_HOVERCAR`, `VARIANT_TRANSPOTTER`, `VARIANT_HOVERBOARD`, `VARIANT_SKATEBOARD`.

## Architecture

### Layered Design

```
main.c            – Main loop (~200 Hz), variant-specific input handling, motor enable/disable
control.c         – High-level command routing
comms.c           – Serial communication protocol, debug protocol, runtime parameter editing
util.c            – Input processing, IIR filtering, rate limiting, mixing algorithms
  ↓
BLDC_controller.c – Core FOC algorithm (AUTO-GENERATED from Simulink, do not manually edit)
BLDC_controller_data.c – Motor calibration parameters and sine/cosine lookup tables
bldc.c            – PWM generation, ADC offset calibration, buzzer (runs in ~16 kHz ISR)
  ↓
setup.c           – Timer, ADC, UART, I2C, DMA initialization
Drivers/STM32F1xx_HAL_Driver/ – ST HAL
Drivers/CMSIS/    – ARM CMSIS
```

### Real-Time Loops

- **~16 kHz ISR** (`DMA1_Channel1_IRQHandler` in `bldc.c`): ADC conversion complete → runs FOC algorithm
- **~200 Hz main loop** (`main.c`): input filtering, command routing, telemetry, variant logic

### Dual-Motor Control

Left motor uses TIM8, right motor uses TIM1 (master-slave synchronized). Each motor has an independent FOC state machine. Phase currents are measured via dual ADCs in synchronized mode with DMA.

### Fixed-Point Arithmetic

The controller uses no floating-point. Parameters are Q14 fixed-point (`fixdt(1,16,14)`): `1.0` = `16384`. When tuning parameters in `BLDC_controller_data.c` or `config.h`, use the Fixed-Point Viewer tool or apply the `Q14` scale manually.

### Key Configuration: `Inc/config.h`

This is the single most important file for customization. Sections include:
- **Variant selection** (`#define VARIANT_*`)
- **Control type/mode**: `CTRL_TYP_SEL` (COM/SIN/FOC), `CTRL_MOD_REQ` (OPEN/VLT/SPD/TRQ)
- **Motor limits**: `I_MOT_MAX` (amps), `N_MOT_MAX` (rpm)
- **Battery calibration**: `BAT_CALIB_REAL_VOLTAGE`, `BAT_CALIB_ADC`, `BAT_CELLS`
- **Field weakening**: `FIELD_WEAK_ENA`, `FIELD_WEAK_MAX`
- **Debug serial**: enable USART2 or USART3 for telemetry

### UART Pin Assignment

- **USART2** (left sensor cable): ADC, PPM, iBUS, debug serial — **not 5V tolerant**
- **USART3** (right sensor cable): serial commands, I2C (Nunchuk) — **5V tolerant**

### `BLDC_controller.c` is auto-generated

Do not manually edit `Src/BLDC_controller.c` or `Src/BLDC_controller_data.c`. These are generated from the Simulink model at https://github.com/EFeru/bldc-motor-control-FOC. Algorithmic changes require regenerating from that model.
