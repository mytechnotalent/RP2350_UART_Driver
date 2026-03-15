# Chapter 13: Pseudo-Instructions — What the Assembler Does For You

## Introduction

A pseudo-instruction looks like a real ARM instruction but is actually translated by the assembler into one or more real instructions.  The assembler provides these as a convenience — they let you write simple, readable code while the tool handles the encoding complexity.  This chapter identifies every pseudo-instruction used in our firmware and explains what the assembler actually generates.

## ldr r0, =value — Load Constant

This is the most common pseudo-instruction in our code:

```asm
  ldr   r0, =RESETS_RESET                        // load 32-bit address into r0
```

### What the Assembler Generates

The assembler examines the value and chooses the best real instruction:

**Case 1: Value fits in mov/movw encoding**

```asm
  ldr   r1, =6                                   // assembler may generate: mov r1, #6
  ldr   r1, =0                                   // assembler may generate: mov r1, #0
```

**Case 2: Value needs a literal pool**

```asm
  ldr   r0, =0x40070000                          // assembler generates:
                                                  //   ldr r0, [pc, #offset]
                                                  //   ...
                                                  //   .word 0x40070000  (in literal pool)
```

The assembler places the 32-bit constant in a nearby read-only area (literal pool) and generates a PC-relative load to fetch it.

### Why This Matters

ARM Thumb-2 instructions have limited space for immediate values.  A 32-bit instruction can encode at most a modified 12-bit constant.  Addresses like 0x40070000 cannot be encoded as immediates, so the literal pool is necessary.

## .equ — Define a Constant Symbol

```asm
  .equ UART0_BASE, 0x40070000
```

`.equ` creates a named constant.  Every occurrence of `UART0_BASE` in subsequent code is replaced by 0x40070000 during assembly.  No instruction is generated — this is purely a textual substitution.

When you write:

```asm
  ldr   r0, =UART0_BASE                          // load UART0 base address
```

The assembler sees:

```asm
  ldr   r0, =0x40070000                          // after symbol substitution
```

And then applies the literal pool logic from above.

## .include — Include Another File

```asm
  .include "constants.s"
```

This replaces the `.include` line with the entire contents of `constants.s`.  It is the assembler's version of C's `#include`.  After inclusion, all `.equ` constants from `constants.s` are available in the current file.

## .global — Export a Symbol

```asm
  .global Reset_Handler                          // make symbol visible to linker
```

This marks `Reset_Handler` as a global symbol so the linker can find it.  Without `.global`, symbols are local to the file and cannot be referenced from other files.

## .type — Declare Symbol Type

```asm
  .type Reset_Handler, %function                 // mark as function symbol
```

This tells the assembler (and debugger) that `Reset_Handler` is a function, not data.  For Thumb code, this also ensures the symbol's address has the Thumb bit (bit 0) set when used as a branch target.

## .size — Declare Symbol Size

```asm
  .size Reset_Handler, . - Reset_Handler         // record function size
```

The `.` is the current address.  `. - Reset_Handler` computes the number of bytes from the start of `Reset_Handler` to the current position.  This metadata helps debuggers and analysis tools.

## .word — Emit a 32-bit Constant

```asm
  .word Reset_Handler + 1                        // emit 4 bytes
```

`.word` is not an instruction — it places a raw 32-bit value into the output.  In our vector table, `.word Reset_Handler + 1` places the address of `Reset_Handler` (with Thumb bit set) at the current location in the output file.

## .byte / .hword — Emit Smaller Constants

```asm
  .byte  0x42                                    // emit 1 byte
  .hword 0x0001                                  // emit 2 bytes
```

Used in our `image_def.s` to construct the PICOBIN block byte by byte.

## Pseudo-Instructions vs Directives

| Type | Purpose | Example |
|---|---|---|
| Pseudo-instruction | Generates real instructions | `ldr r0, =value` |
| Directive | Controls assembly process | `.equ`, `.global`, `.section` |

Pseudo-instructions produce machine code.  Directives produce metadata, constants, or control how the assembler operates.

## Summary

The assembler provides pseudo-instructions and directives that simplify your code.  `ldr r0, =value` loads arbitrary constants using the best available encoding.  `.equ` defines named constants.  `.include` inserts other files.  `.global` and `.type` make symbols visible to the linker.  `.word`, `.byte`, and `.hword` emit raw data.  Understanding these tools explains why our code looks clean even though ARM's encoding is restrictive.
