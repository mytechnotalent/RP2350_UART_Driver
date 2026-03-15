# Chapter 16: Bitwise Operations for Hardware Programming

## Introduction

Hardware registers are collections of individual bits, each controlling a different feature.  Configuring hardware means setting, clearing, and testing specific bits without disturbing others.  This chapter teaches every bitwise technique used in our firmware.

## The Bit Manipulation Toolkit

ARM provides four key instructions for bit manipulation:

| Instruction | Operation | Use Case |
|---|---|---|
| `orr` | Bitwise OR | Set bits to 1 |
| `bic` | Bit Clear (AND NOT) | Clear bits to 0 |
| `and` / `ands` | Bitwise AND | Isolate bits |
| `tst` | Test (AND, discard result) | Test bits, set flags |

## Setting a Bit: orr

To set bit N of a register, OR the register with `(1<<N)`:

```asm
  orr   r1, r1, #(1<<11)                         // set bit 11
```

Before:
```
  r1 = 0b...0000 0000 0000
  Bit 11 is 0
```

After:
```
  r1 = 0b...1000 0000 0000
  Bit 11 is 1, all other bits unchanged
```

This is used to enable hardware features.  In our firmware:

```asm
  orr   r1, r1, #(1<<11)                         // set ENABLE bit in CLK_PERI_CTRL
  orr   r1, r1, #(4<<5)                          // set AUXSRC to XOSC
```

## Clearing a Bit: bic

To clear bit N, use BIC (Bit Clear = AND NOT):

```asm
  bic   r1, r1, #(1<<26)                         // clear bit 26
```

Before:
```
  r1 = 0b...0100 0000 ...
  Bit 26 is 1
```

After:
```
  r1 = 0b...0000 0000 ...
  Bit 26 is 0, all other bits unchanged
```

`bic` is equivalent to `and r1, r1, #~(1<<26)` but more readable.  Used to release peripherals from reset:

```asm
  bic   r1, r1, #(1<<6)                          // clear IO_BANK0 reset bit
  bic   r1, r1, #(1<<26)                         // clear UART0 reset bit
```

## Testing a Bit: tst

To check if bit N is set, use TST:

```asm
  tst   r1, #(1<<31)                             // test bit 31 (XOSC STABLE)
  beq   .wait                                    // branch if bit is clear (Z=1)
```

`tst` computes `r1 AND (1<<31)` and sets the condition flags.  If bit 31 is 0, the result is 0, Z flag is set, and `beq` branches.

```asm
  tst   r1, #(1<<6)                              // test bit 6 (IO_BANK0 done)
  beq   .wait                                    // branch if not done
```

## Isolating Bits: ands

```asm
  ands  r5, r5, r6                               // r5 = r5 AND r6, set flags
```

Used when the mask is in a register rather than an immediate.  In UART0_Out:

```asm
  ldr   r6, =32                                  // mask for bit 5 (TX FIFO full)
  ands  r5, r5, r6                               // isolate bit 5
  bne   .UART0_Out_loop                          // loop if TX FIFO full
```

After `ands`, r5 is either 32 (bit 5 was set, FIFO full) or 0 (bit 5 was clear, FIFO has space).

## Masking Multiple Bits

You can set, clear, or test multiple bits at once:

```asm
  ldr   r1, =0xff                                // mask for bits 0-7
  ands  r0, r0, r1                               // keep only lower 8 bits
```

This masks off the upper 24 bits of r0, keeping only the byte that was received from UART.

## The Read-Modify-Write Pattern

Almost every hardware register access follows this pattern:

```asm
  ldr   r0, =REGISTER_ADDRESS                    // load register address
  ldr   r1, [r0]                                 // READ: load current value
  orr   r1, r1, #(1<<N)                          // MODIFY: set bit N
  str   r1, [r0]                                 // WRITE: store modified value
```

Or for clearing:

```asm
  ldr   r0, =REGISTER_ADDRESS                    // load register address
  ldr   r1, [r0]                                 // READ: load current value
  bic   r1, r1, #(1<<N)                          // MODIFY: clear bit N
  str   r1, [r0]                                 // WRITE: store modified value
```

## Shift for Bit Position

The expression `(1<<N)` creates a value with only bit N set:

| Expression | Value | Hex | Binary |
|---|---:|---:|---|
| (1<<0) | 1 | 0x01 | ...0001 |
| (1<<5) | 32 | 0x20 | ...100000 |
| (1<<6) | 64 | 0x40 | ...1000000 |
| (1<<11) | 2048 | 0x800 | ...100000000000 |
| (1<<26) | 67108864 | 0x4000000 | Bit 26 |
| (1<<31) | 2147483648 | 0x80000000 | MSB |

The assembler evaluates `(1<<N)` at assembly time — no runtime shift occurs.

## Shifting Values into Position

Sometimes you need to set a multi-bit field to a specific value:

```asm
  ldr   r1, =3                                   // r1 = 3 (binary: 11)
  lsl   r1, r1, #8                               // r1 = 0x300 (bits 8 and 9 set)
  orr   r1, r1, #1                               // r1 = 0x301 (bit 0 also set)
```

This creates the UARTCR value: UARTEN (bit 0) + TXE (bit 8) + RXE (bit 9) = 0x301.

## Clearing a Bit Field

```asm
  bic   r5, r5, #0x1f                            // clear bits 0-4 (FUNCSEL field)
  orr   r5, r5, #0x05                            // set FUNCSEL to 5
```

First clear the 5-bit field, then set it to the desired value.  This is a modify operation on a multi-bit field.

## Summary

Bitwise operations are the foundation of hardware programming.  `orr` sets bits, `bic` clears bits, `tst` tests bits, `ands` isolates bits.  The read-modify-write pattern reads a hardware register, changes specific bits, and writes it back.  The expression `(1<<N)` creates a mask for bit N.  Every hardware configuration in our firmware uses these operations.
