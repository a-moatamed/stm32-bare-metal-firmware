# STM32L475 Bare-Metal Firmware — Force Sensor + LED Bar + Buzzer

[![Language: C99](https://img.shields.io/badge/Language-C99-blue.svg?style=flat-square&logo=c&logoColor=white)](https://en.wikipedia.org/wiki/C99)
[![Architecture: ARM Cortex-M4](https://img.shields.io/badge/Architecture-ARM%20Cortex--M4-orange.svg?style=flat-square&logo=arm&logoColor=white)](https://developer.arm.com/Processors/Cortex-M4)
[![MCU: STM32L475VGT6](https://img.shields.io/badge/MCU-STM32L475VGT6-002B49.svg?style=flat-square&logo=stmicroelectronics&logoColor=white)](https://www.st.com/en/microcontrollers-microprocessors/stm32l475vg.html)
[![Board: B-L475E-IOT01A](https://img.shields.io/badge/Board-B--L475E--IOT01A-blueviolet.svg?style=flat-square)](https://www.st.com/en/evaluation-tools/b-l475e-iot01a.html)
[![Build: GNU Make](https://img.shields.io/badge/Build-GNU%20Make-brightgreen.svg?style=flat-square&logo=gnu&logoColor=white)](https://www.gnu.org/software/make/)
[![Toolchain: arm-none-eabi-gcc](https://img.shields.io/badge/Toolchain-arm--none--eabi--gcc-informational.svg?style=flat-square)](https://developer.arm.com/downloads/-/gnu-rm)
[![Flash: st-flash](https://img.shields.io/badge/Flashing-st--flash%20(ST--Link)-yellow.svg?style=flat-square)](https://github.com/stlink-org/stlink)


A production-grade, bare-metal C99 firmware engine for the STM32L475VGT6 (ARM Cortex-M4). This repository demonstrates industrial-grade embedded systems engineering by operating directly on silicon primitives, strictly avoiding vendor Hardware Abstraction Layers (HALs), Low-Layer (LL) drivers, and Real-Time Operating Systems (RTOS).

The system implements asynchronous register-level ADC sampling, mathematical sensor linearization, non-blocking UART telemetry, and hardware PWM control within a highly deterministic, cooperatively multitasked architecture.

## 1. Core Engineering Specifications & Capabilities

* **Deterministic Boot & Initialization:** Custom GNU linker script (`link.ld`) and C reset vector handler (`startup.c`). Establishes exact memory layout, copies `.data` to SRAM, and zeroes `.bss` with zero third-party middleware overhead.
* **Direct Register Manipulation:** Full configuration of memory-mapped control and status registers (`RCC`, `GPIO`, `ADC`, `TIM2`, `UART`, `SysTick`) utilizing custom type-safe structures, explicit bitwise masks, and strict volatile access semantics.
* **Non-Blocking Architecture:** Cooperative multitasking scheduled via a 1 ms monotonic SysTick counter. Peripheral polling and hardware sequencing (e.g., ADC voltage regulator stabilization and calibration) are managed via finite state machines to eliminate CPU busy-wait stalling.
* **Lock-Free Concurrency:** A 256-byte circular FIFO ring buffer decouples application logic from the UART hardware transmit register (`TDR`), preventing 115200-baud serial telemetry from throttling the 2 ms real-time control loop.
* **DSP & Sensor Linearization:** Translates the non-linear hyperbolic conductance curve of a piezoresistive FSR402 sensor into a linear physical response. Utilizes an 8-sample rolling average, a quadratic transfer function, and a 30% deadzone floor threshold to reject mechanical resting noise.

## 2. Hardware Architecture & Pin Mapping

**Target Platform:** STMicroelectronics B-L475E-IOT01A Discovery Kit

| Subsystem | Component | MCU Pin | Hardware Register / Operating Mode |
| --- | --- | --- | --- |
| **Force Acquisition** | FSR402 Sensor | `PC5` | `ADC1_IN14`, `GPIO_MODE_ANALOG`, `ASCR` enabled |
| **Acoustic Transducer** | Piezo Buzzer | `PA15` | `TIM2_CH1`, `GPIO_MODE_AF` (AF1), Push-Pull |
| **Visual Feedback** | 10-Segment LED Bar | Mixed `PA`, `PB`, `PD` | General Purpose Output, driven via atomic `BSRR` |
| **Telemetry Link** | USB Virtual COM (TX/RX) | `PB6` / `PB7` | `UART1_TX` / `UART1_RX`, AF7, 115200 Baud, 8N1 |

## 3. Repository Structure

The project is structured to enforce separation of concerns between peripheral drivers, application logic, and system initialization.

```text
.
├── Makefile                  # Strict GNU Make build system (cross-compiler flags, gc-sections)
├── link.ld                   # Linker script specifying Flash/SRAM boundaries and vector table
├── src/                      # Implementation files
│   ├── startup.c             # Reset handler, memory initialization, vector table definition
│   ├── main.c                # Event loop, sensor linearization logic, and ring buffer management
│   ├── adc.c                 # 5-stage asynchronous ADC initialization and sampling state machine
│   ├── timer.c               # TIM2 hardware PWM configuration for acoustic control
│   ├── uart.c                # TXE-polled UART driver draining the software ring buffer
│   ├── gpio.c                # Atomic bit set/reset operations (BSRR) for jitter-free I/O
│   └── syscalls.c            # Minimal POSIX/newlib stubs (_sbrk, _write) for low-footprint snprintf
└── include/                  # Public API and memory-mapped register definitions
    ├── adc.h                 # SAR ADC register structures
    ├── timer.h               # General-Purpose Timer (TIM2) structures
    ├── uart.h                # UART1 register definitions
    ├── gpio.h                # Port, pin, and alternate function mappings
    ├── systick.h             # ARM Cortex-M core SysTick register map
    └── rcc.h                 # Reset and Clock Control configurations

```

## 4. Quality Assurance & Coding Standards

To ensure determinism and reliability, the codebase adheres to strict industrial standards:

* **Compiler Strictness:** Compiled with `-Wall -Wextra -Werror -Wconversion -Wshadow -pedantic` to elevate all warnings to errors and prevent implicit truncations.
* **Dead-Code Elimination:** Utilizes `-ffunction-sections -fdata-sections` and `--gc-sections` during linking to strip unused symbols, minimizing the Flash footprint.
* **No Dynamic Allocation:** The heap is explicitly restricted. All buffers (including the UART telemetry ring buffer) are statically allocated to prevent memory fragmentation and runtime allocation failures.
* **Atomic I/O:** GPIO manipulation strictly relies on the Bit Set/Reset Register (`BSRR`) to guarantee single-cycle, atomic operations, avoiding read-modify-write race conditions.

## 5. Build & Deployment Environment

### Prerequisites

* **Toolchain:** GNU Arm Embedded Toolchain (`arm-none-eabi-gcc` 10.3-2021.10 or later recommended).
* **Build System:** GNU Make.
* **Flashing Utility:** `stlink` tools (`st-flash`).

### Compilation Instructions

```bash
# Clean previous build artifacts
make clean

# Generate production binary (-Os, debugging symbols stripped)
make

# Generate development binary (-O0, -g3 for GDB analysis)
make DEBUG=1

```

### Flashing to Target

Connect the discovery board via the internal ST-LINK USB interface and deploy to Flash origin `0x08000000`:

```bash
make flash

```

*(If `make flash` is unavailable, execute: `st-flash --reset write firmware.bin 0x08000000`)*

## 6. Telemetry & Diagnostics

The firmware continuously streams real-time, non-blocking telemetry over UART. To view the calibrated duty-cycle output, connect to the enumerated Virtual COM port using a serial terminal:

```bash
# Example using 'minicom' on Linux
minicom -D /dev/ttyACM0 -b 115200

# Example using 'screen' on macOS
screen /dev/tty.usbmodem14202 115200

```