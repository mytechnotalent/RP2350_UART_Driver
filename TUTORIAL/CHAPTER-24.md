# Chapter 24: reset_handler.s — The Boot Sequence Line by Line

## Introduction

After the CPU reads the vector table and jumps to `Reset_Handler`, this function orchestrates the entire boot sequence.  It calls each initialization function in the correct order, then branches to `main`.  This chapter walks through every instruction.

## Full Source: reset_handler.s

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

.global Reset_Handler                            // export Reset_Handler symbol
.type Reset_Handler, %function
Reset_Handler:
  bl    Init_Stack                               // initialize MSP/PSP and limits
  bl    Init_XOSC                                // initialize external crystal oscillator
  bl    Enable_XOSC_Peri_Clock                   // enable XOSC peripheral clock
  bl    Init_Subsystem                           // initialize subsystems
  bl    UART_Release_Reset                       // ensure UART0 out of reset
  bl    UART_Init                                // initialize UART0 (pins, baud, enable)
  bl    Enable_Coprocessor                       // enable CP0 coprocessor
  b     main                                     // branch to main loop
.size Reset_Handler, . - Reset_Handler
```

## Line-by-Line Walkthrough

### Directives

```
.global Reset_Handler                            // export Reset_Handler symbol
.type Reset_Handler, %function
```

Exports `Reset_Handler` so the vector table can reference it.  `.type %function` tells the linker this is a function, which matters for proper Thumb interworking.

### Step 1: Stack Initialization

```
  bl    Init_Stack                               // initialize MSP/PSP and limits
```

`bl` (Branch with Link) saves the return address in `lr` and jumps to `Init_Stack` in `stack.s`.  This sets MSP, PSP, MSPLIM, and PSPLIM.

**Why first?**  Every subsequent function call uses the stack (bl pushes a return address conceptually — actually it saves to lr, but nested calls will push lr to the stack).  The stack must be valid before anything else.

### Step 2: Crystal Oscillator

```
  bl    Init_XOSC                                // initialize external crystal oscillator
```

Calls `Init_XOSC` in `xosc.s`.  This configures the external 12 MHz crystal oscillator and waits until it is stable.

**Why second?**  The CPU is running from the internal ring oscillator after reset.  The ring oscillator is imprecise.  UART baud rate depends on a stable clock, so we start the crystal oscillator before configuring any clock-dependent peripherals.

### Step 3: Peripheral Clock

```
  bl    Enable_XOSC_Peri_Clock                   // enable XOSC peripheral clock
```

Calls `Enable_XOSC_Peri_Clock` in `xosc.s`.  This routes the crystal oscillator to the peripheral clock domain and enables CLK_PERI.

**Why third?**  The peripheral clock drives UART0. Without this step, UART registers would not be clocked and any access would hang or return garbage.

### Step 4: GPIO Subsystem

```
  bl    Init_Subsystem                           // initialize subsystems
```

Calls `Init_Subsystem` in `reset.s`.  This releases IO_BANK0 from reset by clearing bit 6 in RESETS_RESET and polling RESETS_RESET_DONE.

**Why fourth?**  UART pins (GPIO 0 and GPIO 1) use IO_BANK0.  We must release IO_BANK0 from reset before configuring any GPIO function.

### Step 5: UART Reset Release

```
  bl    UART_Release_Reset                       // ensure UART0 out of reset
```

Calls `UART_Release_Reset` in `uart.s`.  This releases UART0 from reset by clearing bit 26 in RESETS_RESET and polling RESETS_RESET_DONE.

**Why fifth?**  The UART peripheral itself is held in reset at power-on.  We must release it before writing any UART registers.

### Step 6: UART Initialization

```
  bl    UART_Init                                // initialize UART0 (pins, baud, enable)
```

Calls `UART_Init` in `uart.s`.  This configures GPIO pins, pad controls, baud rate divisors, line control, and enables UART0.

**Why sixth?**  All prerequisites are met: clock is stable, IO_BANK0 is out of reset, UART0 is out of reset.  Now we can safely configure the UART.

### Step 7: Coprocessor Enable

```
  bl    Enable_Coprocessor                       // enable CP0 coprocessor
```

Calls `Enable_Coprocessor` in `coprocessor.s`.  This sets bits in CPACR to grant access to coprocessor 0, followed by DSB and ISB barriers.

**Why seventh?**  The GPIO coprocessor instructions (mcrr) require CP0 access to be enabled.  While our UART echo loop does not use GPIO coprocessor instructions, the firmware enables CP0 for completeness.

### Step 8: Branch to Main

```
  b     main                                     // branch to main loop
```

`b` (Branch) — not `bl` — jumps to `main`.  There is no `bl` because `main` never returns.  Using `b` instead of `bl` means the link register is not modified, and the stack frame from Reset_Handler is abandoned.

### Size Directive

```
.size Reset_Handler, . - Reset_Handler
```

Records the function size in the ELF symbol table.  The linker and debugger use this to know where Reset_Handler ends.  `. - Reset_Handler` computes the difference between the current address and the start of the function.

## The Boot Sequence Diagram

```
  Power On
    |
    v
  Boot ROM executes
    |
    v
  Boot ROM finds IMAGE_DEF in flash
    |
    v
  CPU reads vector table:
    - MSP = 0x20082000
    - PC  = Reset_Handler
    |
    v
  Reset_Handler
    |
    +---> Init_Stack            (stack.s)
    |       set MSP, PSP, MSPLIM, PSPLIM
    |
    +---> Init_XOSC             (xosc.s)
    |       configure crystal, wait for stable
    |
    +---> Enable_XOSC_Peri_Clock (xosc.s)
    |       route XOSC to peripheral clock
    |
    +---> Init_Subsystem        (reset.s)
    |       release IO_BANK0 from reset
    |
    +---> UART_Release_Reset    (uart.s)
    |       release UART0 from reset
    |
    +---> UART_Init             (uart.s)
    |       configure pins, baud, enable UART
    |
    +---> Enable_Coprocessor    (coprocessor.s)
    |       enable CP0 access
    |
    +---> main                  (main.s)
            echo loop (never returns)
```

## Why Order Matters

Each step depends on the previous one completing:
1. Stack must be valid before any function call
2. Crystal must be started before routing it as a clock source
3. Peripheral clock must be enabled before accessing peripherals
4. IO_BANK0 must be out of reset before configuring GPIO pins
5. UART0 must be out of reset before configuring UART registers
6. UART must be fully configured before the main loop uses it

Reordering any of these steps would cause the firmware to hang or produce incorrect behavior.

## Summary

`Reset_Handler` is the boot orchestrator.  It calls seven initialization functions in a carefully ordered sequence, then branches to `main`.  Each call has a dependency that requires it to run after the previous call.  The `b main` at the end is a one-way branch — the firmware enters the echo loop and never returns.
