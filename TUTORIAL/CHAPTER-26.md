# Chapter 26: reset.s — Releasing IO_BANK0 from Reset

## Introduction

After power-on, most peripherals on the RP2350 are held in reset.  Software must explicitly release each peripheral it wants to use.  The `reset.s` file releases IO_BANK0 — the GPIO function-select controller — from reset.  Without this step, GPIO pin configuration would fail.

## Full Source: reset.s

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

.global Init_Subsystem
.type Init_Subsystem, %function
Init_Subsystem:
.GPIO_Subsystem_Reset:
  ldr   r0, =RESETS_RESET                        // load RESETS->RESET address
  ldr   r1, [r0]                                 // read RESETS->RESET value
  bic   r1, r1, #(1<<6)                          // clear IO_BANK0 bit
  str   r1, [r0]                                 // store value into RESETS->RESET address
.GPIO_Subsystem_Reset_Wait:
  ldr   r0, =RESETS_RESET_DONE                   // load RESETS->RESET_DONE address
  ldr   r1, [r0]                                 // read RESETS->RESET_DONE value
  tst   r1, #(1<<6)                              // test IO_BANK0 reset done
  beq   .GPIO_Subsystem_Reset_Wait               // wait until done
  bx    lr                                       // return
```

## Line-by-Line Walkthrough

### Function Declaration

```
.global Init_Subsystem
.type Init_Subsystem, %function
Init_Subsystem:
```

Exports `Init_Subsystem` as a global function.  Called from `Reset_Handler` after the clock is configured.

### Local Label

```
.GPIO_Subsystem_Reset:
```

A descriptive local label marking the start of the reset-release sequence.  The leading dot makes it local to this file.

### Step 1: Read the Reset Register

```
  ldr   r0, =RESETS_RESET                        // load RESETS->RESET address
  ldr   r1, [r0]                                 // read RESETS->RESET value
```

RESETS_RESET is at address 0x40020000.  This is a 32-bit register where each bit controls one peripheral.  A bit value of 1 means that peripheral is held in reset.  A bit value of 0 means it is released.

At power-on, most bits are 1 (peripherals in reset).

### Step 2: Clear IO_BANK0 Bit

```
  bic   r1, r1, #(1<<6)                          // clear IO_BANK0 bit
```

`bic` (Bit Clear) clears bit 6 in r1.  Bit 6 corresponds to IO_BANK0 in the reset register.  Clearing this bit tells the reset controller to release IO_BANK0.

The read-modify-write pattern (ldr, bic, str) ensures we only modify bit 6 without disturbing the reset state of other peripherals.

### Step 3: Write Back

```
  str   r1, [r0]                                 // store value into RESETS->RESET address
```

Writes the modified value back.  The reset controller begins releasing IO_BANK0, but this is not instantaneous — it takes some clock cycles for the peripheral to come out of reset.

### Step 4: Poll Reset Done

```
.GPIO_Subsystem_Reset_Wait:
  ldr   r0, =RESETS_RESET_DONE                   // load RESETS->RESET_DONE address
  ldr   r1, [r0]                                 // read RESETS->RESET_DONE value
  tst   r1, #(1<<6)                              // test IO_BANK0 reset done
  beq   .GPIO_Subsystem_Reset_Wait               // wait until done
  bx    lr                                       // return
```

RESETS_RESET_DONE is at address 0x40020008.  Each bit reads as 1 when the corresponding peripheral has completed its reset release.

The polling loop:
1. `ldr r1, [r0]` — reads RESET_DONE
2. `tst r1, #(1<<6)` — tests bit 6 (IO_BANK0) by ANDing with 0x40
3. `beq .GPIO_Subsystem_Reset_Wait` — if zero (not done), loop
4. `bx lr` — when bit 6 is 1 (done), return

## The RP2350 Reset Controller

The reset controller manages 32 peripherals.  Key bit assignments:

```
  Bit   Peripheral
  ----  ----------
  0     ADC
  1     BUSCTRL
  2     DMA
  5     I2C0
  6     IO_BANK0    <-- our target
  7     IO_QSPI
  10    PIO0
  11    PIO1
  17    SPI0
  18    SPI1
  22    TIMER0
  24    TRNG
  26    UART0       <-- released separately in uart.s
  27    UART1
  28    USBCTRL
```

Our firmware releases two peripherals from reset:
- IO_BANK0 (bit 6) in `reset.s`
- UART0 (bit 26) in `uart.s`

## Why Not Release Everything at Once?

We could clear multiple bits simultaneously:

```
  bic   r1, r1, #(1<<6)                          // IO_BANK0
  bic   r1, r1, #(1<<26)                         // UART0
  str   r1, [r0]
```

But our design keeps the releases in separate functions for clarity: `Init_Subsystem` handles GPIO, `UART_Release_Reset` handles UART0.  Each function is responsible for one subsystem.

## The Atomic Clear Alternative

The RP2350 provides atomic set/clear/XOR aliases for many registers.  RESETS_RESET_CLEAR at 0x40023000 is the atomic clear alias:

```
  ldr   r0, =RESETS_RESET_CLEAR
  ldr   r1, =(1<<6)
  str   r1, [r0]                                 // atomically clear bit 6
```

Writing a 1 to bit 6 of the atomic clear register clears bit 6 in RESETS_RESET without needing a read-modify-write.  Our firmware uses the read-modify-write approach, which is equally correct for single-core bare-metal code.

## Summary

`Init_Subsystem` releases IO_BANK0 from reset using a read-modify-write on RESETS_RESET, then polls RESETS_RESET_DONE until the release is complete.  This pattern — clear the reset bit, poll the done bit — is the standard way to bring any RP2350 peripheral out of reset.  The same pattern appears in `uart.s` for UART0.
