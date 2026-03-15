# Chapter 9: ARM Arithmetic and Logic Instructions

## Introduction

Arithmetic and logic instructions are the computation engine of the processor.  They take values from registers (and sometimes immediates), perform an operation, and write the result to a destination register.  This chapter covers every arithmetic and logic instruction used in our firmware.

## Arithmetic Instructions

### add — Addition

```asm
  add   r0, r0, #0x04                            // r0 = r0 + 4
```

Adds the immediate 4 to r0 and stores the result in r0.  In our firmware, this computes pad register addresses: `PADS_BANK0_BASE + 0x04`.

### sub / subs — Subtraction

```asm
  subs  r5, r5, #1                               // r5 = r5 - 1, update flags
```

Subtracts 1 from r5.  The `s` suffix means the condition flags are updated:
- Z flag is set if r5 becomes zero
- N flag is set if r5 becomes negative

This is used in our delay loop: decrement counter, branch if not zero.

### mul — Multiplication

```asm
  mul   r5, r0, r4                               // r5 = r0 × r4
```

Multiplies r0 by r4 and stores the result in r5.  In our delay function, this computes the total loop count: `milliseconds × 3600`.

### cmp — Compare

```asm
  cmp   r0, #0                                   // compute r0 - 0, set flags
```

`cmp` is subtraction that discards the result but keeps the flags.  If r0 equals 0, the Z flag is set.  The subsequent `ble .done` branches if r0 ≤ 0.

## Logic Instructions

### and / ands — Bitwise AND

```asm
  ands  r5, r5, r6                               // r5 = r5 AND r6, set flags
```

Each bit of the result is 1 only if the corresponding bits in both operands are 1.  Used to isolate a single bit from a register value:

```
  r5 = 0b...0010 0101   (some status value)
  r6 = 0b...0010 0000   (mask for bit 5 only)
  AND = 0b...0010 0000   (bit 5 preserved, all others zeroed)
```

If the result is zero, the Z flag is set — meaning bit 5 was not set.

### tst — Test Bits

```asm
  tst   r1, #(1<<31)                             // AND r1 with bit 31 mask, set flags
```

`tst` is `and` that discards the result but sets the condition flags.  Used to test whether a specific bit is set without modifying the register:

```
  r1 = 0x80000000 (bit 31 set, XOSC stable)
  tst r1, #(1<<31) → result = 0x80000000, Z=0 (not zero, bit is set)

  r1 = 0x00000000 (bit 31 clear, XOSC not stable)
  tst r1, #(1<<31) → result = 0x00000000, Z=1 (zero, bit is not set)
```

### orr — Bitwise OR

```asm
  orr   r1, r1, #(1<<11)                         // set bit 11 of r1
```

Each bit of the result is 1 if the corresponding bit in either operand is 1.  Used to set specific bits without affecting others:

```
  r1 = 0b...0000 0000 0000   (bit 11 clear)
  imm = 0b...1000 0000 0000   (bit 11 mask)
  OR  = 0b...1000 0000 0000   (bit 11 set, others unchanged)
```

### bic — Bit Clear (AND NOT)

```asm
  bic   r1, r1, #(1<<26)                         // clear bit 26 of r1
```

`bic` computes `r1 AND (NOT immediate)`.  It clears the specified bits while preserving all others:

```
  r1  = 0b...0100 0000 0000 ...   (bit 26 set)
  NOT(1<<26) = 0b...1011 1111 1111 ...
  AND = 0b...0000 0000 0000 ...   (bit 26 cleared)
```

This is the complement of `orr`.  Where `orr` sets bits, `bic` clears them.

### eor — Exclusive OR

Not used in our firmware, but included for completeness.  Each bit is 1 if the two corresponding input bits differ.  Used to toggle bits.

## Shift Instructions

### lsl — Logical Shift Left

```asm
  lsl   r1, r1, #8                               // r1 = r1 << 8
```

Shifts all bits left by 8 positions.  Zeros fill from the right.  Each left shift by 1 doubles the value.

In our UART init, this shifts the RXE/TXE mask into the correct bit positions:

```
  r1 = 0b...0000 0011            (value 3)
  r1 << 8 = 0b...0011 0000 0000  (value 0x300, bits 8 and 9 set)
```

## The Suffix 's' — Flag Updates

Adding `s` to an instruction name causes it to update the condition flags:

| Instruction | Flags Updated | Example |
|---|---|---|
| `add` | No | `add r0, r0, #4` |
| `adds` | Yes | `adds r0, r0, #4` |
| `sub` | No | `sub r0, r0, #1` |
| `subs` | Yes | `subs r5, r5, #1` |
| `and` | No | `and r0, r0, r1` |
| `ands` | Yes | `ands r5, r5, r6` |

The flags are used by subsequent conditional branch instructions (`beq`, `bne`, `ble`, etc.).

## The Read-Modify-Write Pattern

Combining load, logic, and store creates the fundamental hardware configuration pattern:

```asm
  ldr   r0, =RESETS_RESET                        // address
  ldr   r1, [r0]                                 // READ
  bic   r1, r1, #(1<<6)                          // MODIFY: clear bit 6
  str   r1, [r0]                                 // WRITE
```

```asm
  ldr   r0, =CLK_PERI_CTRL                       // address
  ldr   r1, [r0]                                 // READ
  orr   r1, r1, #(1<<11)                         // MODIFY: set bit 11
  orr   r1, r1, #(4<<5)                          // MODIFY: set AUXSRC bits
  str   r1, [r0]                                 // WRITE
```

Nearly every hardware peripheral configuration in our firmware follows this exact pattern.

## Summary

Arithmetic instructions (`add`, `sub`, `mul`, `cmp`) perform math on register values.  Logic instructions (`and`, `orr`, `bic`, `tst`) manipulate individual bits.  The `s` suffix enables condition flag updates for conditional branching.  Shifts (`lsl`) move bits to the correct positions in hardware registers.  The combination of load, logic operation, and store is the universal pattern for configuring hardware peripherals on ARM.
