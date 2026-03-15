# Chapter 29: main.s — The Echo Loop

## Introduction

The `main.s` file is the top-level application.  After `Reset_Handler` completes all hardware initialization, it branches to `main`, which runs an infinite loop: read a byte from UART, send it back.  This simple echo loop proves that every piece of the firmware works correctly.

## Full Source: main.s

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

.global main                                     // export main
.type main, %function                            // mark as function
main:
.Push_Registers:
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
.Loop:
  bl    UART0_In                                 // call UART0_In
  bl    UART0_Out                                // call UART0_Out
  b     .Loop                                    // loop forever
.Pop_Registers:
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return to caller

.section .rodata                                 // read-only data section

.section .data                                   // data section

.section .bss                                    // BSS section
```

## Line-by-Line Walkthrough

### Assembly Preamble

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"
```

Standard preamble plus the constants include.  Even though `main.s` does not directly use any hardware addresses, the include provides the `.syntax`, `.cpu`, and `.thumb` directives that the assembler needs.

### Section and Alignment

```
.section .text                                   // code section
.align 2                                         // align to 4-byte boundary
```

Places the code in the `.text` section, aligned to a 4-byte boundary.

### Function Declaration

```
.global main                                     // export main
.type main, %function                            // mark as function
main:
```

Exports `main` so `Reset_Handler` can branch to it with `b main`.  The `.type %function` marks it as a function for the linker and debugger.

### Save Registers

```
.Push_Registers:
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
```

Saves all callee-saved registers and the link register.  Even though `main` never returns (the loop is infinite), this is good practice: if `main` were ever changed to return, the caller's state would be preserved.

The push decrements the stack pointer and stores 10 registers (r4, r5, r6, r7, r8, r9, r10, r11, r12, lr) = 40 bytes.

### The Echo Loop

```
.Loop:
  bl    UART0_In                                 // call UART0_In
  bl    UART0_Out                                // call UART0_Out
  b     .Loop                                    // loop forever
```

Three instructions form the entire application logic:

1. **`bl UART0_In`** — Branch with Link to `UART0_In`.  This function blocks (polls) until a byte arrives on the RX wire, then returns with the byte in r0

2. **`bl UART0_Out`** — Branch with Link to `UART0_Out`.  The byte is still in r0 (UART0_In returns it there, and bl does not modify r0).  UART0_Out polls until the TX FIFO has space, then writes the byte

3. **`b .Loop`** — Unconditional branch back to the beginning.  `b` (not `bl`) does not save a return address because we never intend to continue past this point

The echo effect: any character typed on the host terminal travels to the RP2350 via the RX wire, is read by `UART0_In`, and is immediately sent back via `UART0_Out` on the TX wire.  The terminal displays the echoed character.

### Why r0 Passes Through

The AAPCS calling convention specifies:
- r0 is used for the first parameter and the return value
- `UART0_In` returns the byte in r0
- `UART0_Out` takes the byte to send in r0

No additional register manipulation is needed between the two calls.  The byte flows from input to output through r0 naturally.

### Dead Code: Pop and Return

```
.Pop_Registers:
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return to caller
```

These instructions are unreachable — the infinite `b .Loop` above never falls through.  They exist for two reasons:
1. Structural completeness: every push should have a matching pop
2. Future-proofing: if the loop were changed to a conditional loop, the function would return correctly

### Empty Data Sections

```
.section .rodata                                 // read-only data section
.section .data                                   // data section
.section .bss                                    // BSS section
```

These three empty sections declare that `main.s` has no read-only data, no initialized data, and no uninitialized data.  They are placeholders — if you added a string constant, it would go in `.rodata`; if you added a global variable with an initial value, it would go in `.data`; if you added a global buffer, it would go in `.bss`.

## The Complete Execution Flow

```
  Host Terminal                    RP2350
  -------------                    ------
  User types 'A'
       |
       v
  Terminal sends 0x41 ---------> UART0 RX FIFO
                                       |
                                 UART0_In polls RXFE
                                 RXFE = 0 (data ready)
                                 r0 = 0x41
                                       |
                                 UART0_Out polls TXFF
                                 TXFF = 0 (space available)
                                 writes 0x41 to UARTDR
                                       |
  Terminal receives 0x41 <------- UART0 TX FIFO
       |
       v
  Terminal displays 'A'
```

## What Makes This a Complete Firmware

Despite being only three functional instructions (bl, bl, b), `main.s` represents a complete bare-metal application because:

1. **It has a well-defined entry point** — Reset_Handler branches here after initialization
2. **It never returns** — bare-metal firmware runs forever
3. **It exercises real hardware** — reading and writing actual peripheral registers
4. **It demonstrates the full stack** — clocks, resets, GPIO, UART, and the CPU all working together

## Summary

`main.s` contains the simplest possible UART application: an infinite loop that reads one byte and echoes it back.  The byte passes through r0 without any additional manipulation, exploiting the AAPCS calling convention.  The empty `.rodata`, `.data`, and `.bss` sections are placeholders for future use.  This three-instruction loop validates that every initialization step completed correctly.
