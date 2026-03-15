# Chapter 7: ARM Cortex-M33 ISA Overview

## Introduction

An Instruction Set Architecture (ISA) defines the contract between software and hardware.  It specifies what instructions the CPU understands, what registers are available, how memory is accessed, and how the CPU responds to exceptions.

ARM Cortex-M33 implements the ARMv8-M architecture with the Thumb-2 instruction set.  This chapter surveys the complete instruction set used in our firmware.

## The ARM Design Philosophy

ARM follows these principles:

1. **Load-store**: only load/store instructions access memory
2. **Fixed register file**: 16 general-purpose registers (r0-r15)
3. **Condition flags**: a status register (xPSR) tracks N, Z, C, V flags
4. **Compact encoding**: Thumb-2 mixes 16-bit and 32-bit instructions for code density
5. **Rich addressing**: multiple addressing modes for flexible memory access
6. **Licensed cores**: ARM designs the core, chip makers (Raspberry Pi) integrate it

## Thumb-2 Instruction Encoding

ARM Cortex-M33 uses Thumb-2, which supports two instruction widths:

### 16-bit (Narrow) Instructions

Compact, common operations.  Limited register access (often r0-r7 only):

```asm
  mov   r0, r1                                   // 16-bit: copy r1 to r0
  add   r0, r0, r1                               // 16-bit: r0 = r0 + r1
  push  {r4-r7, lr}                              // 16-bit: save registers
  bx    lr                                        // 16-bit: return
```

### 32-bit (Wide) Instructions

Full capability: any register, larger immediates, more operations:

```asm
  ldr   r0, =0x40070000                          // 32-bit: load constant
  bic   r1, r1, #(1<<26)                         // 32-bit: clear bit 26
  orr   r1, r1, #(1<<11)                         // 32-bit: set bit 11
  str   r1, [r0, #0x30]                          // 32-bit: store with offset
```

The assembler automatically chooses the narrowest encoding that can represent your instruction.  You write the same syntax either way.

## Instruction Categories

### Data Processing

These instructions operate on registers and immediate values:

| Instruction | Operation | Example |
|---|---|---|
| mov | Copy value | `mov r1, #6` |
| add | Addition | `add r0, r0, r1` |
| sub | Subtraction | `sub r0, r0, #1` |
| mul | Multiplication | `mul r5, r0, r4` |
| and | Bitwise AND | `ands r5, r5, r6` |
| orr | Bitwise OR | `orr r1, r1, #(1<<11)` |
| bic | Bit Clear (AND NOT) | `bic r1, r1, #(1<<26)` |
| eor | Exclusive OR | `eor r0, r0, r1` |
| lsl | Logical Shift Left | `lsl r1, r1, #8` |
| tst | Test (AND, set flags, discard result) | `tst r1, #(1<<31)` |
| cmp | Compare (SUB, set flags, discard result) | `cmp r0, #0` |

### Memory Access

| Instruction | Operation | Example |
|---|---|---|
| ldr | Load word | `ldr r1, [r0]` |
| str | Store word | `str r1, [r0]` |
| ldrb | Load byte | `ldrb r1, [r0]` |
| strb | Store byte | `strb r1, [r0]` |
| push | Push registers | `push {r4-r12, lr}` |
| pop | Pop registers | `pop {r4-r12, lr}` |

### Branch

| Instruction | Operation | Example |
|---|---|---|
| b | Unconditional branch | `b .Loop` |
| beq | Branch if equal (Z=1) | `beq .wait` |
| bne | Branch if not equal (Z=0) | `bne .loop` |
| bl | Branch with link (call) | `bl UART_Init` |
| bx | Branch exchange | `bx lr` |
| ble | Branch if less or equal | `ble .done` |

### System

| Instruction | Operation | Example |
|---|---|---|
| msr | Write special register | `msr MSP, r0` |
| mrs | Read special register | `mrs r0, MSP` |
| dsb | Data synchronization barrier | `dsb` |
| isb | Instruction synchronization barrier | `isb` |
| mcrr | Coprocessor register transfer | `mcrr p0, #4, r2, r4, c4` |

## Instruction Encoding Formats

ARM Thumb-2 instructions have several encoding formats.  The assembler handles encoding, but understanding the format explains why some immediates work and others do not.

### Immediate Values

ARM Thumb-2 32-bit instructions can encode immediates using a modified constant scheme.  The immediate must fit one of these patterns:
- An 8-bit value, optionally rotated by an even amount
- A pattern like 0x00XY00XY, 0xXY00XY00, or 0xXYXYXYXY

If your constant does not fit, use `ldr r0, =value` instead (pseudo-instruction with literal pool).

### Shift Immediates

Shift amounts are encoded directly in the instruction:

```asm
  lsl   r1, r1, #8                               // shift left by 8 bits
```

The shift amount (8) is encoded as a 5-bit field in the instruction word.

## Instructions Used in Our Firmware

Here is every instruction that appears in our actual source files:

```asm
  ldr   r0, =UART0_BASE                          // pseudo-instruction: load 32-bit constant
  ldr   r1, [r0]                                 // load word from memory
  str   r1, [r0]                                 // store word to memory
  str   r1, [r0, #0x30]                          // store with immediate offset
  bic   r1, r1, #(1<<26)                         // bit clear (AND NOT)
  orr   r1, r1, #(1<<11)                         // bitwise OR
  tst   r1, #(1<<31)                             // test bit (AND, set flags)
  ands  r5, r5, r6                               // AND with flag update
  beq   .wait_loop                               // branch if zero flag set
  bne   .poll_loop                               // branch if zero flag clear
  bl    UART_Init                                // branch with link (function call)
  bx    lr                                        // branch to link register (return)
  b     main                                     // unconditional branch
  push  {r4-r12, lr}                             // save registers to stack
  pop   {r4-r12, lr}                             // restore registers from stack
  add   r0, r0, #0x04                            // add immediate
  mov   r1, #6                                   // move immediate to register
  cmp   r0, #0                                   // compare register to immediate
  ble   .done                                    // branch if less or equal
  mul   r5, r0, r4                               // multiply
  subs  r5, r5, #1                               // subtract with flag update
  msr   MSP, r0                                  // write special register
  dsb                                            // data sync barrier
  isb                                            // instruction sync barrier
  mcrr  p0, #4, r2, r4, c4                       // coprocessor transfer
```

## Summary

ARM Cortex-M33 uses the Thumb-2 instruction set, which mixes 16-bit and 32-bit instructions for efficiency.  Instructions fall into four categories: data processing, memory access, branch, and system.  Our firmware uses approximately 20 distinct instructions to configure hardware, poll status registers, and implement a UART echo loop.  The assembler automatically selects the most compact encoding for each instruction.
