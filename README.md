# Embedded Showcase

Index of my embedded firmware and embedded Linux projects. Each links out to its own repo.

## Hardware

| Board                                                                                                                                    | Role                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| STM32F429I Discovery (Cortex-M4F @ 180 MHz, 2 MB flash, 256 KB RAM, 8 MB SDRAM, 2.4" LCD + STMPE811 touch, L3GD20 gyro, onboard ST-LINK) | Main MCU target                                                                                                |
| Raspberry Pi 5                                                                                                                           | Host side of every link, test harness, embedded Linux target                                                   |
| Arduinos                                                                                                                                 | External I2C/SPI devices and misbehaving peripherals for robustness tests (keep the STM32 side at 3.3 V logic) |
| Multimeter                                                                                                                               | Current measurement through the Discovery's IDD jumper                                                         |
| _Recommended:_ 8-channel logic analyzer (sigrok / PulseView)                                                                             | Bus captures for every driver write-up                                                                         |

**Toolchain:** `arm-none-eabi-gcc` + CMake, OpenOCD + GDB, CMSIS device headers for register access. No vendor IDE required to build or flash anything.

## Projects

| Project                                                             | Description                                                                                                                                                                                                                         | Status |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| [stm32-gesture-key](https://github.com/XhovaniM8/stm32-gesture-key) | Gesture-based lock on the STM32F429I Discovery: L3GD20 gyro over SPI with 128-sample bias calibration, moving-average filtering, silence trimming, and per-axis Pearson correlation against a recorded key. Touchscreen UI, Mbed OS | Done   |

## Fundamentals

What each fundamental means in practice, and where it gets proven.

| Fundamental                        | Mastery looks like                                                                                                                                            | Proven in             |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| **Bring-up**                       | Writes the startup code, vector table, and linker script; explains `.data` copy, `.bss` zero, and the reset sequence; configures the clock tree (PLL) by hand | 01                    |
| **Register-level programming**     | Drives peripherals from the reference manual alone; correct use of `volatile`, read-modify-write, and bit fields                                              | 01–04                 |
| **Interrupts**                     | NVIC priorities and preemption; short ISRs; shared data protected without disabling interrupts longer than necessary; ISR latency measured                    | 02, 03                |
| **Timers**                         | Timebase, PWM generation, input capture                                                                                                                       | 03                    |
| **Serial protocols**               | UART, SPI, I2C drivers that are interrupt- or DMA-driven, with error and timeout handling                                                                     | 02, 04                |
| **DMA**                            | Peripheral-to-memory streams, circular and double buffering, handoff to the CPU without tearing                                                               | 04, 06                |
| **ISR-safe data structures**       | Lock-free SPSC ring buffer between ISR and main context                                                                                                       | 02                    |
| **Fault handling and debug**       | HardFault handler that decodes CFSR/HFSR and the stacked PC; MPU stack guard; GDB on real faults                                                              | 05                    |
| **RTOS**                           | Tasks, queues, mutexes vs semaphores, priority inversion (and the fix), stack sizing from high-water marks                                                    | 06                    |
| **Communication protocols**        | Framing (COBS), CRC, sequence numbers, ack/retransmit, parser hardened against garbage input                                                                  | 07                    |
| **Bootloader and firmware update** | Image validation, VTOR relocation and jump to the application, update over a link, rollback on a bad image                                                    | 08                    |
| **Low power**                      | Sleep and Stop modes, wake sources, current measured before and after                                                                                         | 09                    |
| **Testing**                        | Host-side unit tests behind a hardware abstraction, CI, hardware-in-the-loop tests driven by the Pi 5                                                         | 10                    |
| **Embedded Linux**                 | Kernel module, device tree overlay, character device interface to real hardware                                                                               | 11                    |
| **Signal processing**              | Fixed-point filtering and feature extraction on sensor data with measured cycle cost                                                                          | stm32-gesture-key, 12 |

## Definition of done

Every project ships with:

- Builds and flashes from a clean clone with one command; no IDE
- Flash and RAM footprint (`arm-none-eabi-size`)
- Timing measured with the DWT cycle counter (ISR latency, loop and task times) where it applies
- Host-side unit tests for any logic that doesn't touch registers
- Logic analyzer capture of the bus traffic for driver projects
- Failure handling documented: timeouts, bus errors, bad input
- Demo video
- Write-up: design decisions, what broke and why, what I'd change

## Roadmap

Ordered so each project builds on the ones before it. Drivers written in Phase 1 are reused for everything after.

### Phase 1: Bare metal (STM32F429I Discovery, no HAL)

| #   | Project           | Builds                                                                                                                                                                                                                   | Fundamentals                                           |
| --- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ |
| 01  | baremetal-bringup | Own startup file, vector table, and linker script; clock tree to max frequency; SysTick timebase; GPIO blink and button EXTI. Debug the reset path in GDB                                                                | Bring-up, Register-level                               |
| 02  | uart-driver       | Interrupt-driven UART TX/RX on SPSC ring buffers; `printf` retargeting; overrun and framing-error handling. The Pi 5's GPIO UART is the other end                                                                        | Interrupts, Serial protocols, ISR-safe data structures |
| 03  | timers-pwm        | PWM (LED dimming, servo), input capture for frequency measurement, ISR latency measured with DWT                                                                                                                         | Timers, Interrupts                                     |
| 04  | spi-i2c-dma       | SPI driver with DMA for the onboard L3GD20 gyro; I2C driver for the onboard STMPE811 touch controller; bus-error and timeout recovery tested against a misbehaving Arduino I2C device                                    | Serial protocols, DMA                                  |
| 05  | fault-handling    | HardFault handler that dumps CFSR, HFSR, fault addresses, and the stacked PC over UART; deliberately triggered faults (bad pointer, unaligned access, divide-by-zero trap, stack overflow caught by an MPU guard region) | Fault handling and debug                               |

### Phase 2: RTOS and systems (STM32 + Pi 5)

| #   | Project              | Builds                                                                                                                                                                                                                   | Fundamentals                   |
| --- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------ |
| 06  | freertos-sensor-node | FreeRTOS tasks for DMA sensor sampling, filtering, comms, and a CLI; queues between tasks; priority inversion shown with a binary semaphore and fixed with a mutex; stack high-water marks; per-task runtime stats       | RTOS, DMA                      |
| 07  | host-protocol        | Framed binary protocol between the STM32 and the Pi 5: COBS framing, CRC32, sequence numbers, ack/retransmit. Python host tool on the Pi; parser fuzzed with random and corrupted frames                                 | Communication protocols        |
| 08  | bootloader           | Bootloader in the first 32 KB of flash: CRC-validates the application image, relocates VTOR, sets the MSP, jumps. Firmware update over UART using the protocol from 07, with rollback to the known-good image on failure | Bootloader and firmware update |
| 09  | low-power            | Sleep and Stop modes with EXTI and RTC wakeup; current measured through the IDD jumper in each mode; duty-cycled sensor node                                                                                             | Low power                      |

### Phase 3: Testing and embedded Linux

| #   | Project         | Builds                                                                                                                                                                                                                                           | Fundamentals               |
| --- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------- |
| 10  | test-harness    | Driver logic behind a hardware abstraction, unit tested on the host; GitHub Actions builds firmware and runs host tests on every push. Hardware-in-the-loop: the Pi 5 flashes the STM32 with OpenOCD and runs pytest suites over UART and GPIO   | Testing                    |
| 11  | pi-linux-driver | The STM32 acts as an SPI peripheral with a register map; a Linux kernel module on the Pi 5 exposes it as a character device, configured with a device tree overlay                                                                               | Embedded Linux             |
| 12  | gesture-key-v2  | Rebuild stm32-gesture-key on my own drivers + FreeRTOS: DMA sampling at a higher output data rate, fixed-point filtering with CMSIS-DSP, key stored in flash, matching time and power measured. Before/after comparison against the Mbed version | Capstone: all of the above |

### Optional

| Project          | Builds                                                                                                                                         | Extra hardware                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| usb-cdc          | USB CDC device on the onboard USB OTG port (TinyUSB); replaces the UART link for 07                                                            | None                                |
| fmc-ltdc-display | FMC init of the onboard 8 MB SDRAM, a framebuffer in SDRAM, the LCD driven by LTDC with DMA2D for fills and blits; replaces the gesture-key UI | None                                |
| can-bus          | bxCAN driver; STM32 ↔ Arduino network with filtering, error frames, and bus-off recovery                                                       | 2 CAN transceivers + MCP2515 module |
