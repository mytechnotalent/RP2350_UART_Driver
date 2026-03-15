# Chapter 18: The RP2350 Microcontroller — Architecture and Hardware

## Introduction

This chapter describes the RP2350 chip itself: its processor cores, memory subsystem, peripheral interconnect, reset and clock infrastructure, and boot process.  Understanding the hardware is essential before walking through the firmware that configures it.

## RP2350 Block Diagram

```
                         RP2350
                         ======

   +--------------+    +--------------+
   |     ARM      |    |   RISC-V     |   dual architecture
   |  Cortex-M33  |    |   Hazard3    |   (choose at boot)
   |    Core 0    |    |    Core 0    |
   +------+-------+    +------+-------+
          |                   |
          +--------+----------+
                   |
          +--------+---------+
          |   Bus Fabric     |
          | (AHB-Lite xbar)  |
          +--+-----+------+--+
             |     |      |
        +----+  +--+--+  +--+---+
        |       |     |  |      |
   +----+---+   +--+--+  +---+--+---+
   |  SRAM  |   |Flash|  |   APB    |
   |  520K  |   | XIP |  |  Bridge  |
   +--------+   +-----+  +----+-----+
                              |
              +---------------+---------------+
              |       APB Peripherals         |
              |  UART  SPI  I2C  GPIO  PWM    |
              |  Timers  ADC  Clocks  Resets  |
              +-------------------------------+
```

## The ARM Cortex-M33 Core

Our firmware uses the ARM Cortex-M33 core.  Key features:

- **Architecture**: ARMv8-M Mainline
- **Instruction set**: Thumb-2 (mixed 16/32-bit)
- **Registers**: 16 general-purpose (r0-r15)
- **Clock**: up to 150 MHz
- **Pipeline**: 3-stage (fetch, decode, execute)
- **Hardware divide**: single-cycle UDIV/SDIV
- **Coprocessor interface**: supports custom coprocessors (GPIO on RP2350)
- **TrustZone**: hardware security (not used in our firmware)
- **NVIC**: nested vectored interrupt controller

## Memory Map

The ARM Cortex-M33 has a 4 GB address space divided into regions:

| Region | Address Range | Size | Contents |
|---|---|---|---|
| Code | 0x00000000 - 0x0FFFFFFF | 256 MB | Boot ROM, flash alias |
| XIP | 0x10000000 - 0x11FFFFFF | 32 MB | External flash (execute-in-place) |
| SRAM | 0x20000000 - 0x20081FFF | 520 KB | On-chip RAM |
| Peripherals | 0x40000000 - 0x4FFFFFFF | 256 MB | APB peripheral registers |
| PPB | 0xE0000000 - 0xE00FFFFF | 1 MB | Private peripheral bus |

## The Bus Fabric

The RP2350 uses an AHB-Lite crossbar switch to connect the CPU to memory and peripherals.  The bus fabric routes each memory access to the correct target:

- Addresses 0x10xxxxxx → flash (XIP cache)
- Addresses 0x20xxxxxx → SRAM
- Addresses 0x40xxxxxx → APB bridge → peripherals
- Addresses 0xE0xxxxxx → PPB (system control)

The APB (Advanced Peripheral Bus) bridge converts AHB transactions to the slower APB protocol used by peripherals.  This is why peripheral register reads may take multiple clock cycles.

## The Reset Controller

The RP2350 reset controller holds most peripherals in reset at power-on.  Before you can use a peripheral, you must release it from reset and wait for the reset sequence to complete:

```asm
  ldr   r0, =RESETS_RESET                        // reset control register
  ldr   r1, [r0]                                 // read current value
  bic   r1, r1, #(1<<6)                          // clear IO_BANK0 reset bit
  str   r1, [r0]                                 // write back

  ldr   r0, =RESETS_RESET_DONE                   // reset done status register
  ldr   r1, [r0]                                 // read status
  tst   r1, #(1<<6)                              // test IO_BANK0 done bit
  beq   .wait                                    // loop until done
```

Key reset bits:
- Bit 6 → IO_BANK0 (GPIO function selection)
- Bit 26 → UART0

## The Clock System

At power-on, the RP2350 runs from an internal ring oscillator (~6 MHz).  For accurate UART timing, we switch to the external crystal oscillator (XOSC) at 12 MHz.

The clock initialization sequence:
1. Configure XOSC startup delay
2. Enable XOSC with the correct frequency range
3. Wait for XOSC to stabilize (bit 31 of XOSC_STATUS)
4. Set the peripheral clock to use XOSC as its source

## The XOSC (Crystal Oscillator)

The RP2350 expects a 12 MHz crystal on the board.  The Raspberry Pi Pico 2 provides this.  XOSC registers:

| Register | Address | Purpose |
|---|---|---|
| XOSC_CTRL | 0x40048000 | Enable and frequency range |
| XOSC_STATUS | 0x40048004 | Stable flag (bit 31) |
| XOSC_STARTUP | 0x4004800C | Startup delay counter |

## UART0 Peripheral

UART (Universal Asynchronous Receiver/Transmitter) provides serial communication.  The RP2350's UART is a PrimeCell PL011-compatible design by ARM.

Two signals:
- **TX** (transmit): GPIO0 on the Pico 2
- **RX** (receive): GPIO1 on the Pico 2

Data format: 8 data bits, no parity, 1 stop bit (8N1) at 115200 baud.

The UART has 32-byte FIFOs for both transmit and receive.  Our firmware polls the flag register to check FIFO status before writing or reading data.

## GPIO and Pin Multiplexing

Each GPIO pin on the RP2350 can serve multiple functions.  The IO_BANK0 peripheral controls which function is connected to each pin:

- GPIO0_CTRL FUNCSEL = 2 → UART0 TX
- GPIO1_CTRL FUNCSEL = 2 → UART0 RX

The PADS_BANK0 peripheral controls electrical characteristics: pull-up/down, drive strength, input enable, output disable.

## The Coprocessor Interface

ARM Cortex-M33 supports up to 8 coprocessors (CP0-CP7).  The RP2350 uses CP0 for fast GPIO access via the `mcrr` instruction:

```asm
  mcrr  p0, #4, r2, r4, c4                       // GPIO coprocessor operation
```

This bypasses the APB bus for faster GPIO control.  The CPACR register must be configured to enable coprocessor access before using `mcrr`.

## The Boot Process

When power is applied:
1. Boot ROM runs from address 0x00000000
2. Boot ROM scans flash for a valid IMAGE_DEF block
3. Boot ROM finds the vector table and reads the initial SP and reset vector
4. CPU jumps to Reset_Handler
5. Reset_Handler initializes stack, clocks, peripherals, then enters main

The IMAGE_DEF block in `image_def.s` tells the bootrom that this is an ARM binary (not RISC-V) and that it should boot in secure mode.

## The Vector Table

ARM Cortex-M33 requires a vector table in memory.  The first two entries are:

| Offset | Content | Purpose |
|---|---|---|
| 0x00 | Initial SP | Stack pointer value loaded at reset |
| 0x04 | Reset vector | Address of Reset_Handler (with Thumb bit) |

Additional entries (not used in our firmware) would hold addresses for NMI, HardFault, and other exception handlers.

## Summary

The RP2350 is a dual-architecture microcontroller.  Our firmware uses the ARM Cortex-M33 core with Thumb-2 instructions.  The chip has 520 KB SRAM, up to 16 MB external flash, and a rich peripheral set connected through an AHB-Lite bus fabric and APB bridge.  The clock system starts from an internal oscillator and switches to a 12 MHz external crystal.  The reset controller holds peripherals in reset until software releases them.  The boot sequence starts from ROM, finds the IMAGE_DEF block, reads the vector table, and jumps to our Reset_Handler.
