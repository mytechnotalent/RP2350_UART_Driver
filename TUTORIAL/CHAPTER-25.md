# Chapter 25: xosc.s — Crystal Oscillator Initialization

## Introduction

The RP2350 has an internal ring oscillator that runs at roughly 12 MHz, but it drifts with temperature and voltage.  UART communication requires precise timing.  The external crystal oscillator (XOSC) provides a stable 12 MHz clock source.  The `xosc.s` file contains two functions: one to start the crystal and one to route it as the peripheral clock.

## Full Source: xosc.s

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

.global Init_XOSC
.type Init_XOSC, %function
Init_XOSC:
  ldr   r0, =XOSC_STARTUP                        // load XOSC_STARTUP address
  ldr   r1, =0x00c4                              // set delay 50,000 cycles
  str   r1, [r0]                                 // store value into XOSC_STARTUP
  ldr   r0, =XOSC_CTRL                           // load XOSC_CTRL address
  ldr   r1, =0x00FABAA0                          // set 1_15MHz, freq range, actual 14.5MHz
  str   r1, [r0]                                 // store value into XOSC_CTRL
.Init_XOSC_Wait:
  ldr   r0, =XOSC_STATUS                         // load XOSC_STATUS address
  ldr   r1, [r0]                                 // read XOSC_STATUS value
  tst   r1, #(1<<31)                             // test STABLE bit
  beq   .Init_XOSC_Wait                          // wait until stable bit is set
  bx    lr                                       // return

.global Enable_XOSC_Peri_Clock
.type Enable_XOSC_Peri_Clock, %function
Enable_XOSC_Peri_Clock:
  ldr   r0, =CLK_PERI_CTRL                       // load CLK_PERI_CTRL address
  ldr   r1, [r0]                                 // read CLK_PERI_CTRL value
  orr   r1, r1, #(1<<11)                         // set ENABLE bit
  orr   r1, r1, #(4<<5)                          // set AUXSRC: XOSC_CLKSRC bit
  str   r1, [r0]                                 // store value into CLK_PERI_CTRL
  bx    lr                                       // return
```

## Function 1: Init_XOSC

### Purpose

Starts the external crystal oscillator at the 1–15 MHz frequency range and waits until it is stable.

### Step 1: Configure Startup Delay

```
  ldr   r0, =XOSC_STARTUP                        // load XOSC_STARTUP address
  ldr   r1, =0x00c4                              // set delay 50,000 cycles
  str   r1, [r0]                                 // store value into XOSC_STARTUP
```

XOSC_STARTUP is at address 0x4004800C.  The value 0x00C4 (decimal 196) sets the startup delay.  The hardware multiplies this by 256, giving 196 x 256 = 50,176 ring-oscillator cycles.  This gives the crystal time to stabilize before the STABLE bit is set.

The crystal does not start oscillating instantly.  It takes thousands of cycles for the mechanical resonance to build up to a stable amplitude.  The startup delay prevents software from using the oscillator before it is ready.

### Step 2: Enable the Oscillator

```
  ldr   r0, =XOSC_CTRL                           // load XOSC_CTRL address
  ldr   r1, =0x00FABAA0                          // set 1_15MHz, freq range, actual 14.5MHz
  str   r1, [r0]                                 // store value into XOSC_CTRL
```

XOSC_CTRL is at address 0x40048000.  The value 0x00FABAA0 encodes:

```
  Bits 23:12 = 0xFAB  -> ENABLE: magic value to enable XOSC
  Bits 11:0  = 0xAA0  -> FREQ_RANGE: 1_15MHZ range
```

The ENABLE field requires the specific value 0xFAB (not just a single bit) as a safety mechanism — random memory corruption is unlikely to produce this exact pattern.  The FREQ_RANGE selects the 1–15 MHz range, matching our 12 MHz crystal.

### Step 3: Wait for Stable

```
.Init_XOSC_Wait:
  ldr   r0, =XOSC_STATUS                         // load XOSC_STATUS address
  ldr   r1, [r0]                                 // read XOSC_STATUS value
  tst   r1, #(1<<31)                             // test STABLE bit
  beq   .Init_XOSC_Wait                          // wait until stable bit is set
  bx    lr                                       // return
```

XOSC_STATUS is at address 0x40048004.  Bit 31 (STABLE) is set by hardware once the startup delay has elapsed and the oscillator is running.

The polling loop:
1. `ldr r1, [r0]` — reads the status register
2. `tst r1, #(1<<31)` — tests bit 31 by ANDing r1 with 0x80000000 and setting flags
3. `beq .Init_XOSC_Wait` — if the result is zero (STABLE not set), loop back
4. `bx lr` — once STABLE is set, return

The label `.Init_XOSC_Wait` has a leading dot, making it a local label.  It is only visible within this file.

## Function 2: Enable_XOSC_Peri_Clock

### Purpose

Routes the crystal oscillator to the peripheral clock domain and enables CLK_PERI.

### Step 1: Read CLK_PERI_CTRL

```
  ldr   r0, =CLK_PERI_CTRL                       // load CLK_PERI_CTRL address
  ldr   r1, [r0]                                 // read CLK_PERI_CTRL value
```

CLK_PERI_CTRL is at address 0x40010048.  We read the current value first because we want to modify specific bits without disturbing others (read-modify-write pattern).

### Step 2: Set ENABLE and AUXSRC

```
  orr   r1, r1, #(1<<11)                         // set ENABLE bit
  orr   r1, r1, #(4<<5)                          // set AUXSRC: XOSC_CLKSRC bit
```

Two `orr` operations modify the read value:
- `orr r1, r1, #(1<<11)` sets bit 11 (ENABLE), turning on the peripheral clock
- `orr r1, r1, #(4<<5)` sets bits 7:5 to 4 (binary 100).  AUXSRC = 4 selects XOSC_CLKSRC as the clock source

The AUXSRC field selects which oscillator feeds the peripheral clock:
- 0 = CLK_SYS
- 1 = PLL_SYS
- 2 = PLL_USB
- 3 = ROSC_CLKSRC_PH
- 4 = XOSC_CLKSRC (our selection)

### Step 3: Write Back

```
  str   r1, [r0]                                 // store value into CLK_PERI_CTRL
  bx    lr                                       // return
```

Writes the modified value back to CLK_PERI_CTRL and returns.  After this function, the peripheral clock (CLK_PERI) is running from the 12 MHz crystal oscillator.

## Clock Path Diagram

```
  12 MHz Crystal
       |
       v
  XOSC (0x40048000)
  - STARTUP delay: 196 x 256 cycles
  - CTRL: enable + 1-15MHz range
  - STATUS: poll STABLE bit
       |
       v
  CLK_PERI_CTRL (0x40010048)
  - AUXSRC = 4 (XOSC)
  - ENABLE = 1
       |
       v
  UART0 peripheral clock
  - Used for baud rate generation
  - IBRD = 6, FBRD = 33
  - Baud = 12MHz / (16 * (6 + 33/64)) = ~115200
```

## Why XOSC Matters for UART

UART communication requires both sides to agree on the baud rate within about 3% tolerance.  The internal ring oscillator can drift by 10–20%, which would cause framing errors.  The crystal oscillator is accurate to within tens of parts per million (ppm), ensuring reliable UART communication.

## Summary

`Init_XOSC` starts the crystal oscillator by configuring the startup delay, writing the enable pattern and frequency range to XOSC_CTRL, and polling the STABLE bit.  `Enable_XOSC_Peri_Clock` routes the stable crystal clock to the peripheral clock domain using a read-modify-write on CLK_PERI_CTRL.  Together, these two functions provide the stable clock that UART0 needs for accurate baud rate generation.
