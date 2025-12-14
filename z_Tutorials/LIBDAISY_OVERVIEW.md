# libDaisy Submodule Overview

## Introduction

libDaisy is the Hardware Abstraction Library (HAL) for the Daisy Audio Platform developed by Electrosmith. It provides a comprehensive C++ interface to access hardware features of the STM32H750 microcontroller and external peripherals used on Daisy boards.

**Version**: v5.4.0+  
**License**: MIT  
**MCU Target**: STM32H750IB (Cortex-M7, 480MHz, 128KB Flash, 1MB+ SRAM, 64MB SDRAM)  
**Official Documentation**: [https://electro-smith.github.io/libDaisy](https://electro-smith.github.io/libDaisy)  
**Repository**: [https://github.com/electro-smith/libDaisy](https://github.com/electro-smith/libDaisy)

## Project Structure

libDaisy organizes its codebase into several key directories with specific naming conventions:

### Directory Layout

```
libDaisy/
├── core/               # Build system and linker scripts
├── src/                # Main source code
│   ├── daisy_*.h/cpp   # Board-specific APIs (Seed, Patch, Pod, etc.)
│   ├── dev/            # External device drivers
│   ├── hid/            # Human Interface Devices
│   ├── per/            # MCU peripherals
│   ├── sys/            # System-level configuration
│   ├── ui/             # User interface building blocks
│   ├── usbd/           # USB Device mode
│   ├── usbh/           # USB Host mode
│   └── util/           # Internal library utilities
├── Drivers/            # STM32 HAL/CMSIS drivers
├── Middlewares/        # Third-party middleware (USB, FatFs)
├── examples/           # Example projects
├── tests/              # Unit tests
└── doc/                # Documentation
```

### Naming Conventions (Prefixes)

| Prefix | Meaning | Example |
|--------|---------|---------|
| **sys** | System-level configuration (clocks, DMA, etc.) | `system.h`, `dma.h` |
| **per** | MCU peripherals (SPI, I2C, UART, ADC, etc.) | `spi.h`, `i2c.h`, `adc.h` |
| **dev** | External device drivers (codecs, displays, sensors) | `codec_ak4556.h`, `oled_ssd130x.h`, `sdram.h` |
| **hid** | Human Interface Devices (switches, LEDs, encoders) | `switch.h`, `encoder.h`, `led.h` |
| **ui** | User interface components (menus, event queues) | UI building blocks |
| **util** | Internal library utilities (not exposed via daisy.h) | Internal helpers |
| **daisy** | Core API files and board-specific implementations | `daisy_seed.h`, `daisy_patch_sm.h` |

## Core Components

### 1. Board Support (src/)

Pre-configured classes for specific Daisy hardware platforms:

- **`daisy_seed.h/cpp`** - Daisy Seed and Seed2 DFM (the core module)
- **`daisy_patch_sm.h/cpp`** - Patch SM (compact eurorack module)
- **`daisy_patch.h/cpp`** - Patch (full-size eurorack module)
- **`daisy_pod.h/cpp`** - Pod (desktop synthesizer)
- **`daisy_petal.h/cpp`** - Petal (guitar pedal)
- **`daisy_field.h/cpp`** - Field (expressive touch controller)
- **`daisy_versio.h/cpp`** - Versio (eurorack DSP platform)
- **`daisy_legio.h/cpp`** - Legio (eurorack oscillator platform)

Each board class initializes the appropriate hardware configuration (audio codec, GPIO mappings, controls, etc.) for that platform.

### 2. Audio (hid/audio.h)

Audio engine providing:
- Configurable sample rates (8kHz - 96kHz)
- Configurable block sizes (1 - 256 samples)
- Stereo input/output
- Audio callback mechanism
- Support for multiple codecs (AK4556, PCM3060, WM8731)

**Example**:
```cpp
hw.Init();
hw.SetAudioBlockSize(4);  // 4 samples per callback
hw.SetAudioSampleRate(SaiHandle::Config::SampleRate::SAI_48KHZ);
hw.StartAudio(AudioCallback);

void AudioCallback(AudioHandle::InputBuffer in, 
                   AudioHandle::OutputBuffer out, 
                   size_t size) {
    for(size_t i = 0; i < size; i++) {
        out[0][i] = in[0][i];  // passthrough
        out[1][i] = in[1][i];
    }
}
```

### 3. Peripherals (per/)

Low-level MCU peripheral drivers:

| Peripheral | File | Purpose |
|------------|------|---------|
| ADC | `adc.h` | Analog-to-digital conversion (knobs, CV inputs) |
| DAC | `dac.h` | Digital-to-analog conversion (CV outputs) |
| GPIO | `gpio.h` | General purpose I/O pins |
| I2C | `i2c.h` | Inter-IC communication bus |
| SPI | `spi.h` | Serial Peripheral Interface |
| SAI | `sai.h` | Serial Audio Interface (for audio codecs) |
| UART | `uart.h` | Serial communication (MIDI, debugging) |
| QSPI | `qspi.h` | Quad-SPI (external flash memory) |
| SDMMC | `sdmmc.h` | SD card interface |
| TIM | `tim.h` | Hardware timers |
| RNG | `rng.h` | Random number generator |

### 4. Human Interface Devices (hid/)

High-level interface components:

- **`switch.h`** - Debounced switches/buttons (with press, release, hold detection)
- **`encoder.h`** - Rotary encoders (incremental and detented)
- **`led.h`** - Single LEDs with PWM brightness control
- **`rgb_led.h`** - RGB LEDs
- **`ctrl.h`** - Analog controls (knobs, sliders, CV inputs) with filtering
- **`gatein.h`** - Gate/trigger inputs
- **`midi.h`** - MIDI input/output (UART and USB)
- **`usb.h`** - USB communication (Audio, MIDI, Serial)
- **`logger.h`** - Debug logging system
- **`parameter.h`** - Parameter mapping and scaling

### 5. External Devices (dev/)

Drivers for external components:

**Audio Codecs**:
- `codec_ak4556.h` - AK4556 stereo codec (Daisy Seed, Patch SM)
- `codec_pcm3060.h` - PCM3060 stereo codec
- `codec_wm8731.h` - WM8731 stereo codec (Daisy Pod, Petal, Patch)

**Memory**:
- `sdram.h` - 64MB SDRAM (IS42S16160G)
- `flash_IS25LP064A.h` - 64Mbit QSPI flash
- `flash_IS25LP080D.h` - 8Mbit QSPI flash

**Displays**:
- `oled_ssd130x.h` - SSD1306/SSD1309 OLED displays (I2C/SPI)
- `lcd_hd44780.h` - HD44780 character LCD

**LED Drivers**:
- `leddriver.h` - LED matrix drivers
- `neopixel.h` - WS2812B addressable LEDs
- `dotstar.h` - APA102 addressable LEDs
- `sr_595.h` - 74HC595 shift register (LED expansion)

**Sensors**:
- `apds9960.h` - Gesture/proximity/color sensor
- `dps310.h` - Barometric pressure sensor
- `icm20948.h` - 9-axis IMU (accel, gyro, magnetometer)
- `tlv493d.h` - 3D magnetic sensor

**I/O Expansion**:
- `mcp23x17.h` - MCP23017/MCP23S17 I2C/SPI GPIO expander
- `mpr121.h` - MPR121 capacitive touch sensor
- `neotrellis.h` - NeoTrellis keypad
- `max11300.h` - MAX11300 programmable analog/digital I/O
- `sr_4021.h` - 74HC4021 shift register (input expansion)

### 6. System (sys/)

Core system configuration:

- **`system.h`** - System initialization, clock configuration, reset control
- **`dma.h`** - Direct Memory Access configuration
- **`fatfs.h`** - FAT filesystem support (for SD cards)
- **`stm32h7xx_hal_conf.h`** - STM32 HAL configuration

### 7. Build System (core/)

- **`Makefile`** - Generic Makefile for easy project setup
- **Linker Scripts**:
  - `STM32H750IB_flash.lds` - Standard flash boot (128KB internal flash)
  - `STM32H750IB_qspi.lds` - Boot from QSPI flash (8MB external)
  - `STM32H750IB_sram.lds` - Boot from SRAM (development/testing)
- **Bootloader Binaries**: DFU bootloaders with various timeout configurations
- **`startup_stm32h750xx.c/s`** - MCU startup code and vector table

## Memory Regions (Linker Script)

The STM32H750 provides multiple memory regions configured in the linker script:

| Region | Base Address | Size | Purpose |
|--------|--------------|------|---------|
| **FLASH** | 0x08000000 | 128 KB | Internal flash (code and read-only data) |
| **DTCMRAM** | 0x20000000 | 128 KB | Tightly-coupled RAM (fast, for stack/data) |
| **SRAM** | 0x24000000 | 480 KB | AXI SRAM (general use) |
| **RAM_D2** | 0x30000000 | 288 KB | AHB SRAM (DMA, heap) |
| **RAM_D3** | 0x38000000 | 64 KB | AHB SRAM (backup domain) |
| **ITCMRAM** | 0x00000000 | 64 KB | Instruction TCM RAM |
| **SDRAM** | 0xC0000000 | 64 MB | External SDRAM (large buffers) |
| **QSPIFLASH** | 0x90000000 | 8 MB | External QSPI flash (XIP or storage) |
| **BACKUP_SRAM** | 0x38800000 | 4 KB | Battery-backed SRAM |

### SDRAM Usage

Large audio buffers (delay lines, reverbs, loopers) should be placed in SDRAM using the `DSY_SDRAM_BSS` macro:

```cpp
#include "dev/sdram.h"

float DSY_SDRAM_BSS large_buffer[48000 * 10];  // 10 seconds at 48kHz
```

The SDRAM controller is initialized automatically by board `Init()` methods.

## Key Features

### Audio Processing
- **Low-latency audio** - Block sizes as small as 1 sample (20.8μs @ 48kHz)
- **Multiple sample rates** - 8kHz to 96kHz
- **Stereo I/O** - 24-bit audio via high-quality codecs
- **Floating-point processing** - Hardware FPU acceleration

### Control Inputs
- **Analog inputs** - 12-bit ADC with oversampling and filtering
- **Digital inputs** - Debounced switches, encoders, gates
- **MIDI** - UART and USB MIDI with parser
- **CV inputs** - 0-5V or ±5V (board-dependent)

### Communication
- **USB** - Audio class, MIDI, CDC (serial), MSC (mass storage)
- **UART** - Serial communication (up to 4 ports)
- **I2C** - Multiple I2C buses for sensors/peripherals
- **SPI** - Multiple SPI buses

### Storage
- **SD Card** - FatFs filesystem support
- **QSPI Flash** - 8MB external flash for samples/presets
- **Persistent Storage** - Non-volatile settings storage

## Common Usage Patterns

### Basic Project Setup

```cpp
#include "daisy_patch_sm.h"

using namespace daisy;
using namespace patch_sm;

DaisyPatchSM hw;

void AudioCallback(AudioHandle::InputBuffer in,
                   AudioHandle::OutputBuffer out,
                   size_t size) {
    for(size_t i = 0; i < size; i++) {
        out[0][i] = in[0][i];
        out[1][i] = in[1][i];
    }
}

int main(void) {
    hw.Init();
    hw.StartAudio(AudioCallback);
    while(1) {}
}
```

### Using Controls

```cpp
// Read knob/CV inputs
hw.ProcessAllControls();
float knob1 = hw.GetAdcValue(CV_1);  // 0.0 to 1.0

// Read switch
Switch button;
button.Init(hw.B7);
button.Debounce();
if(button.RisingEdge()) {
    // button was pressed
}
```

### MIDI

```cpp
MidiHandler midi;
midi.Init(MidiHandler::INPUT_MODE_UART1, 
          MidiHandler::OUTPUT_MODE_NONE);
midi.StartReceive();

while(1) {
    midi.Listen();
    while(midi.HasEvents()) {
        auto msg = midi.PopEvent();
        // handle MIDI message
    }
}
```

## Building Projects

### Using the Generic Makefile

```makefile
# Project Name
TARGET = MyProject

# Sources
CPP_SOURCES = MyProject.cpp

# Library Locations
LIBDAISY_DIR = ../../libDaisy
DAISYSP_DIR = ../../DaisySP

# Core location and generic Makefile
SYSTEM_FILES_DIR = $(LIBDAISY_DIR)/core
include $(SYSTEM_FILES_DIR)/Makefile
```

Then build with:
```bash
make clean
make
```

### Flashing Firmware

```bash
# Via USB DFU bootloader
make program-dfu

# Via ST-Link debugger
make program
```

## Hardware Abstraction Benefits

- **Board Independence** - Switch between Daisy platforms with minimal code changes
- **Clean API** - High-level interfaces hide complexity
- **Tested & Stable** - Community-vetted, production-ready
- **Extensible** - Easy to add custom drivers following existing patterns
- **Well-Documented** - Doxygen documentation and examples

## Recent Updates (v5.4.0)

- Official Daisy Seed2 DFM support
- ADC conversion speed configuration
- New Pin system for improved GPIO handling
- SK9822 LED driver
- Improved MIDI UART stability
- QSPI persistent storage cache fix
- Software SPI transport for OLED displays

## Resources

- **Official Docs**: [https://electro-smith.github.io/libDaisy](https://electro-smith.github.io/libDaisy)
- **Forum**: [https://forum.electro-smith.com](https://forum.electro-smith.com)
- **Discord**: [Electrosmith Discord](https://discord.gg/ByHBnMtQTR)
- **Wiki**: [https://github.com/electro-smith/DaisyWiki/wiki](https://github.com/electro-smith/DaisyWiki/wiki)
- **Examples**: `libDaisy/examples/`

## Summary

libDaisy provides a comprehensive, production-ready hardware abstraction layer for the Daisy platform, enabling rapid development of audio applications without low-level MCU programming. Its modular design, extensive peripheral support, and board-specific APIs make it easy to create complex audio processors, synthesizers, and effects with minimal boilerplate code.
