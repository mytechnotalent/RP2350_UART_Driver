# Chapter 22: constants.s — Every .equ Definition Explained

## Introduction

The `constants.s` file defines every memory-mapped register address and constant used throughout the firmware.  Every other source file includes this file with `.include "constants.s"`.  This chapter explains every constant — what it is, why it has that value, and where it is used.

## Full Source: constants.s

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.equ STACK_TOP,                   0x20082000
.equ STACK_LIMIT,                 0x2007a000
.equ XOSC_BASE,                   0x40048000
.equ XOSC_CTRL,                   XOSC_BASE + 0x00
.equ XOSC_STATUS,                 XOSC_BASE + 0x04
.equ XOSC_STARTUP,                XOSC_BASE + 0x0c
.equ PPB_BASE,                    0xe0000000
.equ CPACR,                       PPB_BASE + 0x0ed88
.equ CLOCKS_BASE,                 0x40010000
.equ CLK_PERI_CTRL,               CLOCKS_BASE + 0x48
.equ RESETS_BASE,                 0x40020000
.equ RESETS_RESET,                RESETS_BASE + 0x0
.equ RESETS_RESET_CLEAR,          RESETS_BASE + 0x3000
.equ RESETS_RESET_DONE,           RESETS_BASE + 0x8
.equ IO_BANK0_BASE,               0x40028000
.equ IO_BANK0_GPIO16_CTRL_OFFSET, 0x84
.equ PADS_BANK0_BASE,             0x40038000
.equ PADS_BANK0_GPIO16_OFFSET,    0x44
.equ UART0_BASE,                  0x40070000
```

## Line-by-Line Walkthrough

### Assembly Preamble

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set
```

These three directives appear at the top of every source file.  Since `constants.s` is included into every other file, these directives propagate everywhere:
- `.syntax unified` — enables the unified ARM/Thumb syntax where one instruction set covers both
- `.cpu cortex-m33` — tells the assembler which instructions are valid
- `.thumb` — all instructions are Thumb-2 encoded

### Stack Constants

```
.equ STACK_TOP,                   0x20082000
.equ STACK_LIMIT,                 0x2007a000
```

- **STACK_TOP** = 0x20082000.  This is the initial stack pointer value.  SRAM starts at 0x20000000 and is 512 KB, so the top of SRAM is 0x20080000.  Our stack top is 8 KB above that — in the stack-guard region of the RP2350 memory map
- **STACK_LIMIT** = 0x2007A000.  The lowest address the stack can reach.  STACK_TOP - STACK_LIMIT = 0x8000 = 32 KB of stack space

Used in: `stack.s` (Init_Stack), `vector_table.s` (initial SP in vector table).

### XOSC (External Crystal Oscillator)

```
.equ XOSC_BASE,                   0x40048000
.equ XOSC_CTRL,                   XOSC_BASE + 0x00
.equ XOSC_STATUS,                 XOSC_BASE + 0x04
.equ XOSC_STARTUP,                XOSC_BASE + 0x0c
```

- **XOSC_BASE** = 0x40048000.  Base address of the XOSC peripheral in the RP2350 memory map
- **XOSC_CTRL** = 0x40048000.  Control register at offset 0x00.  Used to set the frequency range and enable the oscillator
- **XOSC_STATUS** = 0x40048004.  Status register at offset 0x04.  Bit 31 (STABLE) indicates the oscillator is ready
- **XOSC_STARTUP** = 0x4004800C.  Startup delay register at offset 0x0C.  Controls how many clock cycles to wait before declaring the XOSC stable

Used in: `xosc.s` (Init_XOSC, Enable_XOSC_Peri_Clock).

### PPB (Private Peripheral Bus) and CPACR

```
.equ PPB_BASE,                    0xe0000000
.equ CPACR,                       PPB_BASE + 0x0ed88
```

- **PPB_BASE** = 0xE0000000.  The ARM Private Peripheral Bus, containing system control registers (NVIC, SysTick, SCB, etc.)
- **CPACR** = 0xE000ED88.  Coprocessor Access Control Register.  Controls which coprocessors (CP0-CP15) are accessible.  Our firmware enables CP0 for GPIO coprocessor instructions

Used in: `coprocessor.s` (Enable_Coprocessor).

### Clocks

```
.equ CLOCKS_BASE,                 0x40010000
.equ CLK_PERI_CTRL,               CLOCKS_BASE + 0x48
```

- **CLOCKS_BASE** = 0x40010000.  Base address of the clock controller
- **CLK_PERI_CTRL** = 0x40010048.  Peripheral clock control register.  Bit 11 enables the peripheral clock, bits 7:5 select the clock source (AUXSRC)

Used in: `xosc.s` (Enable_XOSC_Peri_Clock).

### Reset Controller

```
.equ RESETS_BASE,                 0x40020000
.equ RESETS_RESET,                RESETS_BASE + 0x0
.equ RESETS_RESET_CLEAR,          RESETS_BASE + 0x3000
.equ RESETS_RESET_DONE,           RESETS_BASE + 0x8
```

- **RESETS_BASE** = 0x40020000.  Base address of the reset controller
- **RESETS_RESET** = 0x40020000.  The reset register.  Each bit controls one peripheral.  Writing 1 holds that peripheral in reset; writing 0 releases it
- **RESETS_RESET_CLEAR** = 0x40023000.  Atomic clear alias.  Writing 1 to a bit clears that bit in RESETS_RESET without a read-modify-write
- **RESETS_RESET_DONE** = 0x40020008.  Status register.  A bit reads as 1 when that peripheral has completed its reset sequence

Key bits:
- Bit 6 = IO_BANK0 (GPIO subsystem)
- Bit 26 = UART0

Used in: `reset.s` (Init_Subsystem), `uart.s` (UART_Release_Reset).

### IO_BANK0 (GPIO Function Select)

```
.equ IO_BANK0_BASE,               0x40028000
.equ IO_BANK0_GPIO16_CTRL_OFFSET, 0x84
```

- **IO_BANK0_BASE** = 0x40028000.  Base address of the IO bank that controls GPIO function selection
- **IO_BANK0_GPIO16_CTRL_OFFSET** = 0x84.  The CTRL register offset for GPIO16.  Each GPIO has a STATUS and CTRL register pair at 8-byte intervals

UART uses GPIO 0 and 1, so UART_Init accesses offsets 0x04 (GPIO0_CTRL) and 0x0C (GPIO1_CTRL) directly.  The GPIO16 constant is available for LED or other GPIO use.

Used in: `uart.s` (UART_Init), `gpio.s` (GPIO_Config).

### PADS_BANK0 (Pad Control)

```
.equ PADS_BANK0_BASE,             0x40038000
.equ PADS_BANK0_GPIO16_OFFSET,    0x44
```

- **PADS_BANK0_BASE** = 0x40038000.  Base address for pad control registers (drive strength, pull-up/down, input enable, output disable)
- **PADS_BANK0_GPIO16_OFFSET** = 0x44.  Pad register offset for GPIO16

Used in: `uart.s` (UART_Init), `gpio.s` (GPIO_Config).

### UART0

```
.equ UART0_BASE,                  0x40070000
```

- **UART0_BASE** = 0x40070000.  Base address of the UART0 peripheral (PL011)

Key register offsets used in `uart.s`:
- 0x00 = UARTDR (data register)
- 0x18 = UARTFR (flag register)
- 0x24 = UARTIBRD (integer baud-rate divisor)
- 0x28 = UARTFBRD (fractional baud-rate divisor)
- 0x2C = UARTLCR_H (line control register)
- 0x30 = UARTCR (control register)

Used in: `uart.s` (UART_Init, UART0_Out, UART0_In).

## How .equ Works

`.equ` defines a symbol with a constant value.  It does not emit any bytes into the binary.  When the assembler encounters the symbol name in an instruction, it substitutes the value at assembly time.

```
.equ UART0_BASE, 0x40070000
ldr  r0, =UART0_BASE                            // assembler generates: ldr r0, [pc, #offset]
                                                 // and places 0x40070000 in literal pool
```

This is different from `.word`, which emits bytes.  `.equ` is purely a text substitution — a named constant.

## The .include Mechanism

Every source file contains:

```
.include "constants.s"
```

The assembler physically inserts the contents of `constants.s` at that point.  This means every `.equ` definition and every directive (`.syntax unified`, `.cpu cortex-m33`, `.thumb`) is present in every translation unit.

## Summary

`constants.s` is the single source of truth for all hardware addresses.  It defines the XOSC, clock, reset, GPIO, pad, UART, and system control register addresses.  Every other source file includes it.  If a hardware address changes, only this file needs to be updated.
