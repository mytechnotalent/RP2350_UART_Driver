# Chapter 8: ARM Immediate and Move Instructions

## Introduction

Many instructions need a constant value — a number baked directly into the instruction encoding.  ARM calls these immediate values.  This chapter explains how immediates work on ARM Cortex-M33, what restrictions exist, and how the assembler handles constants that do not fit.

## The mov Instruction

`mov` copies a value into a register.  When the source is an immediate, the value is encoded inside the instruction itself:

```asm
  mov   r1, #6                                   // r1 = 6
```

**What happens**: the processor writes the value 6 into register r1.  The previous contents of r1 are overwritten.

**Encoding**: the 16-bit Thumb encoding of `mov rd, #imm8` can encode immediates from 0 to 255 in an 8-bit field.

### mov with Larger Immediates

For values from 0 to 65535, the assembler uses `movw` (move wide):

```asm
  movw  r1, #1234                                // r1 = 1234 (lower 16 bits)
```

For full 32-bit constants, a `movw`/`movt` pair is needed:

```asm
  movw  r0, #0x0000                              // r0[15:0] = 0x0000
  movt  r0, #0x4007                              // r0[31:16] = 0x4007, r0 = 0x40070000
```

`movt` (move top) writes the upper 16 bits without touching the lower 16 bits.

## The ldr Pseudo-Instruction

When you write `ldr r0, =value`, the assembler decides how to load the constant:

```asm
  ldr   r0, =0x40070000                          // r0 = 0x40070000
```

If the value fits in a `mov` or `movw`, the assembler generates that instruction.  If not, it places the constant in a **literal pool** (a data area near the code) and generates a PC-relative load.  This is transparent to you — the syntax is the same regardless.

### Literal Pool

The literal pool is a small region of read-only data that the assembler places after the current function or at an `.ltorg` directive.  Example of what the assembler might generate:

```
  ; Your code:
  ldr   r0, =0x40070000

  ; Assembler generates:
  ldr   r0, [pc, #offset]        ; load from literal pool
  ...
  .word 0x40070000               ; literal pool entry
```

Every `ldr r0, =CONSTANT` in our firmware works this way when the constant is too large for a `mov` encoding.

## The ldr Immediate Instruction

Do not confuse the pseudo-instruction `ldr r0, =value` with the real instruction `ldr r0, [r1, #offset]`.  The real instruction loads from memory:

```asm
  ldr   r1, [r0, #0x18]                          // r1 = memory[r0 + 0x18]
```

The `#0x18` here is an immediate offset added to the base register r0.  Load offsets can range from 0 to 4095 for word loads.

## Immediate Encoding in Thumb-2

ARM Thumb-2 32-bit data processing instructions use a "modified constant" encoding.  An immediate can be:

1. **An 8-bit value** (0-255): `mov r0, #42`
2. **An 8-bit value rotated right** by an even amount: `mov r0, #0xFF00`
3. **Replicated patterns**: `0x00XY00XY`, `0xXY00XY00`, `0xXYXYXYXY`

If your immediate does not match any of these patterns, the assembler rejects it.  Use `ldr r0, =value` instead.

### Which Immediates Work?

| Immediate | Works in mov? | Why |
|---|---|---|
| #6 | Yes | Fits 8-bit |
| #33 | Yes | Fits 8-bit |
| #112 | Yes | Fits 8-bit (0x70) |
| #0x100 | Yes | 0x01 rotated left 8 |
| #(1<<26) | Yes | 0x04 rotated left 24 |
| #(1<<31) | Yes | 0x80 rotated left 24 |
| #0x40070000 | No | Does not fit any pattern |
| #3600 | No | 0xE10 does not fit |

For values that do not fit, `ldr r0, =value` is the standard solution.

## Constants in Our Firmware

Our firmware uses these loading patterns:

```asm
  ldr   r0, =RESETS_RESET                        // 32-bit address, uses literal pool
  ldr   r1, =2                                   // small constant, may become mov
  ldr   r1, =0x00FABAA0                          // large constant, uses literal pool
  ldr   r1, =6                                   // may become mov r1, #6
  ldr   r1, =33                                  // may become mov r1, #33
  ldr   r1, =112                                 // may become mov r1, #112
  ldr   r4, =3600                                // 3600 = 0xE10, uses literal pool
```

The assembler optimizes each one.  The `ldr r0, =value` syntax is always safe to use — the assembler picks the best encoding.

## The add and sub Immediates

These instructions can encode small constants directly:

```asm
  add   r0, r0, #0x04                            // r0 = r0 + 4
  subs  r5, r5, #1                               // r5 = r5 - 1, update flags
```

The `s` suffix on `subs` means the instruction updates the condition flags (N, Z, C, V).  Without the `s`, the flags are not changed.

## Summary

ARM Cortex-M33 can embed immediate values directly in instructions, but the encoding limits which constants are valid.  Small values (0-255) always work.  Larger values work if they match a rotation or replication pattern.  For arbitrary 32-bit values, `ldr r0, =value` uses a literal pool.  The assembler handles all of this transparently — you just write the instruction and let the tool pick the encoding.
