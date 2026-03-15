# Chapter 2: Number Systems — Binary, Hexadecimal, and Decimal

## Introduction

Every value inside a computer is stored as binary: ones and zeros.  When you write assembly, you constantly work with binary and hexadecimal representations.  This chapter teaches you the three number systems used in embedded programming and how to convert between them fluently.

## Decimal — Base 10

Decimal is the number system you already know.  It uses ten digits: 0 through 9.  Each position represents a power of 10.

```
  Position:    3     2     1     0
  Power:       10³   10²   10¹   10⁰
  Value:       1000  100   10    1

  Example: 4097 = 4×1000 + 0×100 + 9×10 + 7×1
```

Decimal is convenient for humans but inconvenient for hardware.  Hardware thinks in powers of 2.

## Binary — Base 2

Binary uses two digits: 0 and 1.  Each position represents a power of 2.

```
  Position:    7     6     5     4     3     2     1     0
  Power:       2⁷    2⁶    2⁵    2⁴    2³    2²    2¹    2⁰
  Value:       128   64    32    16    8     4     2     1
```

A single binary digit is called a **bit**.  Eight bits form a **byte**.  A byte can represent values from 0 (00000000) to 255 (11111111).

### Binary to Decimal Example

```
  Binary:  10100110
           1×128 + 0×64 + 1×32 + 0×16 + 0×8 + 1×4 + 1×2 + 0×1
           = 128 + 32 + 4 + 2
           = 166
```

### Decimal to Binary Example

```
  Decimal: 42
  42 ÷ 2 = 21 remainder 0
  21 ÷ 2 = 10 remainder 1
  10 ÷ 2 = 5  remainder 0
  5  ÷ 2 = 2  remainder 1
  2  ÷ 2 = 1  remainder 0
  1  ÷ 2 = 0  remainder 1

  Read remainders bottom-to-top: 101010
  Binary: 00101010
```

## Hexadecimal — Base 16

Hexadecimal (hex) uses sixteen digits: 0-9 and A-F.  Each hex digit represents exactly 4 bits.

```
  Hex:    0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
  Dec:    0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
  Bin:    0000 0001 0010 0011 0100 0101 0110 0111
          1000 1001 1010 1011 1100 1101 1110 1111
```

Hex is used everywhere in embedded programming because:
1. Each hex digit maps to exactly 4 bits — no mental math needed
2. A 32-bit value fits in exactly 8 hex digits
3. Memory addresses are naturally expressed in hex

### Hex to Binary Conversion

Convert each hex digit independently to 4 bits:

```
  Hex:    0x4007_0000
          4       0       0       7       0       0       0       0
          0100    0000    0000    0111    0000    0000    0000    0000

  Binary: 0100 0000 0000 0111 0000 0000 0000 0000
```

### Binary to Hex Conversion

Group bits into sets of 4, starting from the right:

```
  Binary: 0001 1010 0000 0000 0000 0000 0000 0000
          1       A       0       0       0       0       0       0
  Hex:    0x1A000000
```

## The 0x Prefix

In assembly and C, hexadecimal numbers are prefixed with `0x`.  Binary numbers are prefixed with `0b`.  Numbers with no prefix are decimal.

```asm
  ldr   r0, =0x40070000                          // hex: UART0 base address
  mov   r1, #112                                 // decimal: 112 = 0x70
  mov   r2, #0b00100000                          // binary: bit 5 set
```

## Bit Numbering

Bits are numbered from the right, starting at 0.  In a 32-bit register:

```
  Bit 31 (MSB) ............................ Bit 0 (LSB)
  [31][30][29]...[8][7][6][5][4][3][2][1][0]
```

- **Bit 0** is the Least Significant Bit (LSB)
- **Bit 31** is the Most Significant Bit (MSB)
- **Bit N** has the value 2^N

When we say "set bit 5", we mean make bit 5 equal to 1.  The value of bit 5 alone is 2⁵ = 32 = 0x20.

## Common Bit Patterns in Our Firmware

| Pattern | Decimal | Hex | Binary | Meaning |
|---|---:|---:|---:|---|
| (1<<0) | 1 | 0x01 | 0000...0001 | Bit 0 (UARTEN) |
| (1<<4) | 16 | 0x10 | 0000...10000 | Bit 4 (RX FIFO empty) |
| (1<<5) | 32 | 0x20 | 0000...100000 | Bit 5 (TX FIFO full) |
| (1<<6) | 64 | 0x40 | 0000...1000000 | Bit 6 (IO_BANK0) |
| (1<<11) | 2048 | 0x800 | 1000 0000 0000 | Bit 11 (CLK enable) |
| (1<<26) | 67108864 | 0x4000000 | Bit 26 | UART0 reset bit |
| (1<<31) | 2147483648 | 0x80000000 | Bit 31 | XOSC stable |

## Two's Complement — Signed Numbers

ARM Cortex-M33 uses two's complement for signed integers.  In a 32-bit register:

- Positive numbers: bit 31 = 0, value is the normal binary value
- Negative numbers: bit 31 = 1, value is computed by inverting all bits and adding 1

```
  +1:  0000 0000 0000 0000 0000 0000 0000 0001
  -1:  1111 1111 1111 1111 1111 1111 1111 1111
  +42: 0000 0000 0000 0000 0000 0000 0010 1010
  -42: 1111 1111 1111 1111 1111 1111 1101 0110
```

Our firmware does not use signed arithmetic, but understanding two's complement is essential for reading hardware status registers where bit 31 is often a flag.

## Data Sizes on ARM Cortex-M33

| Name | Size | Range (unsigned) | ARM suffix |
|---|---|---|---|
| Byte | 8 bits | 0 to 255 | B |
| Halfword | 16 bits | 0 to 65,535 | H |
| Word | 32 bits | 0 to 4,294,967,295 | (default) |

ARM Cortex-M33 is a 32-bit processor.  Each register holds a 32-bit word.  Memory addresses are 32 bits.  Most instructions operate on 32-bit values.

## Summary

Binary is the native language of hardware.  Hexadecimal is the convenient shorthand.  Every hex digit maps to exactly 4 bits.  Bit positions are numbered from 0 (rightmost) to 31 (leftmost).  When we write `(1<<5)` in our firmware, it means the value with only bit 5 set: decimal 32, hex 0x20, binary 00100000.  Mastering these conversions is essential for the rest of this book.
