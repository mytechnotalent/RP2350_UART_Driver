# Chapter 23: stack.s and vector_table.s — Stack Initialization and the Vector Table

## Introduction

Two files work together to get the CPU running after reset: `vector_table.s` provides the hardware's first two words (initial stack pointer and reset address), and `stack.s` provides a function that properly configures all four stack registers.  This chapter explains both files line by line.

## Part 1: vector_table.s

### Full Source

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .vectors, "ax"                          // vector table section
.align 2                                         // align to 4-byte boundary

.global _vectors                                 // export _vectors symbol
_vectors:
  .word STACK_TOP                                // initial stack pointer
  .word Reset_Handler + 1                        // reset handler (Thumb bit set)
```

### Line-by-Line Walkthrough

```
.section .vectors, "ax"                          // vector table section
```

Creates a section named `.vectors` with flags `"ax"`: allocatable and executable.  The linker script places this section at a 128-byte boundary in flash, after the IMAGE_DEF block.

```
.align 2                                         // align to 4-byte boundary
```

Aligns the current position to a 4-byte boundary (2^2 = 4).  The vector table entries are 32-bit words and must be word-aligned.

```
.global _vectors                                 // export _vectors symbol
_vectors:
```

Exports the `_vectors` label so the linker can reference it.  The linker script uses `PROVIDE(__Vectors = ADDR(.vectors))` to create an alias.

```
  .word STACK_TOP                                // initial stack pointer
```

The first word of the vector table (offset 0x00) is the initial Main Stack Pointer value.  When the CPU comes out of reset, the hardware reads this word and loads it into MSP.  STACK_TOP = 0x20082000.

```
  .word Reset_Handler + 1                        // reset handler (Thumb bit set)
```

The second word (offset 0x04) is the reset vector — the address where execution begins.  The `+ 1` sets bit 0, which is the Thumb indicator.  Cortex-M33 always executes in Thumb mode, and the hardware requires bit 0 to be set in vector table entries.  At reset, the CPU reads this word, clears bit 0 to get the actual address, and branches there in Thumb mode.

### What Happens at Power-On

1. The boot ROM runs, finds the IMAGE_DEF block, validates it, and jumps to our binary
2. The CPU reads address 0x10000080 (the vector table base): loads 0x20082000 into MSP
3. The CPU reads address 0x10000084: loads Reset_Handler + 1
4. The CPU strips bit 0 and branches to Reset_Handler in Thumb mode

### Why Only Two Entries?

A full Cortex-M33 vector table has 16 system exceptions plus up to 240 device interrupts.  Our firmware does not use interrupts, so we only define the two mandatory entries: initial SP and reset vector.  If an unhandled exception occurs, the CPU will fault because there is no handler — but for our polling-based UART driver, this is sufficient.

## Part 2: stack.s

### Full Source

```
.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

.global Init_Stack
.type Init_Stack, %function
Init_Stack:
  ldr   r0, =STACK_TOP                           // load stack top
  msr   PSP, r0                                  // set PSP
  ldr   r0, =STACK_LIMIT                         // load stack limit
  msr   MSPLIM, r0                               // set MSP limit
  msr   PSPLIM, r0                               // set PSP limit
  ldr   r0, =STACK_TOP                           // reload stack top
  msr   MSP, r0                                  // set MSP
  bx    lr                                       // return
```

### Line-by-Line Walkthrough

```
.section .text                                   // code section
.align 2                                         // align to 4-byte boundary
```

Places the following code in the `.text` section.  The `.align 2` ensures the first instruction starts on a 4-byte boundary.

```
.global Init_Stack
.type Init_Stack, %function
```

Exports `Init_Stack` as a global function symbol.  `.type %function` marks it as a function for the linker and debugger.

```
Init_Stack:
  ldr   r0, =STACK_TOP                           // load stack top
```

Loads 0x20082000 into r0 using a literal pool load.  The assembler places the constant in the literal pool and generates a PC-relative load.

```
  msr   PSP, r0                                  // set PSP
```

`msr` (Move to Special Register) writes r0 to the Process Stack Pointer.  PSP is the stack pointer used when the CPU is in Thread mode with CONTROL.SPSEL = 1.  Even though our firmware uses MSP (Handler mode uses MSP always), setting PSP provides a valid stack if the mode ever switches.

```
  ldr   r0, =STACK_LIMIT                         // load stack limit
  msr   MSPLIM, r0                               // set MSP limit
  msr   PSPLIM, r0                               // set PSP limit
```

Loads 0x2007A000 into r0, then writes it to both MSPLIM and PSPLIM.  These are Cortex-M33 stack limit registers (new in ARMv8-M).  If the stack pointer goes below this value, the CPU generates a UsageFault with STKOF (stack overflow) — a hardware-enforced stack guard.

```
  ldr   r0, =STACK_TOP                           // reload stack top
  msr   MSP, r0                                  // set MSP
```

Reloads STACK_TOP and writes it to the Main Stack Pointer.  The hardware already set MSP from the vector table, but this ensures MSP is at the expected value after any boot ROM activity that may have modified it.

```
  bx    lr                                       // return
```

Returns to the caller (Reset_Handler).  `bx lr` branches to the address in the link register.

### The Four Stack Registers

```
  Register  Value        Purpose
  --------  -----------  -------
  MSP       0x20082000   Main Stack Pointer (used by exception handlers)
  PSP       0x20082000   Process Stack Pointer (used by thread code)
  MSPLIM    0x2007A000   MSP lower limit (hardware stack overflow guard)
  PSPLIM    0x2007A000   PSP lower limit (hardware stack overflow guard)
```

### Why Init_Stack Exists

The vector table already sets MSP at reset.  But Init_Stack does more:
1. Sets PSP (the vector table only sets MSP)
2. Sets both stack limit registers (the vector table cannot do this)
3. Guarantees MSP is correct regardless of what the boot ROM did

## Summary

`vector_table.s` provides the two words the hardware needs at reset: the initial stack pointer and the reset vector address.  `stack.s` provides Init_Stack, which configures all four stack registers (MSP, PSP, MSPLIM, PSPLIM) for safe operation with hardware stack overflow detection.  Together, they form the foundation that makes all subsequent code execution possible.
