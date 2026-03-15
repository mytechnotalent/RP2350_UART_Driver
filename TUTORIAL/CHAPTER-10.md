# Chapter 10: ARM Memory Access Instructions — Load and Store Deep Dive

## Introduction

On ARM Cortex-M33, only load and store instructions access memory.  This chapter examines every form of memory access instruction used in our firmware, including addressing modes, offsets, and pseudo-instructions.

## ldr — Load Register

### Basic Form

```asm
  ldr   r1, [r0]                                 // r1 = memory[r0]
```

Reads a 32-bit word from the address in r0 into r1.  The address must be word-aligned (divisible by 4).

### Immediate Offset

```asm
  ldr   r1, [r0, #0x18]                          // r1 = memory[r0 + 0x18]
```

Adds the constant 0x18 to r0 to compute the effective address, then loads the word.  The base register r0 is not modified.

This is used throughout our firmware to access different registers within a peripheral block:

```asm
  ldr   r0, =UART0_BASE                          // r0 = 0x40070000 (base)
  ldr   r5, [r0, #0x18]                          // read UARTFR (base + 0x18)
  str   r1, [r0, #0x24]                          // write UARTIBRD (base + 0x24)
  str   r1, [r0, #0x28]                          // write UARTFBRD (base + 0x28)
  str   r1, [r0, #0x2c]                          // write UARTLCR_H (base + 0x2c)
  str   r1, [r0, #0x30]                          // write UARTCR (base + 0x30)
```

With a single base register holding UART0_BASE, we can access any register in the UART peripheral using immediate offsets.

### Pseudo-Instruction Form

```asm
  ldr   r0, =0x40070000                          // r0 = 0x40070000
```

This is NOT a memory access — it is a pseudo-instruction that loads a constant into a register.  The assembler may generate a `mov`, `movw/movt` pair, or a literal pool load depending on the constant.  The `=` sign distinguishes this from a real load.

## str — Store Register

### Basic Form

```asm
  str   r1, [r0]                                 // memory[r0] = r1
```

Writes the 32-bit value in r1 to the address in r0.

### Immediate Offset

```asm
  str   r1, [r0, #4]                             // memory[r0 + 4] = r1
```

Computes the address as r0 + 4 and stores r1 there.

## push and pop — Stack Operations

### push

```asm
  push  {r4-r12, lr}                             // save registers to stack
```

This is equivalent to:
1. Subtract 4 × (number of registers) from SP
2. Store each register to the stack at consecutive addresses

The registers are stored in order from lowest to highest register number, at ascending memory addresses.  After `push {r4-r12, lr}`:

```
  SP - 40: r4
  SP - 36: r5
  SP - 32: r6
  SP - 28: r7
  SP - 24: r8
  SP - 20: r9
  SP - 16: r10
  SP - 12: r11
  SP - 8:  r12
  SP - 4:  lr
  SP now points to SP - 40
```

### pop

```asm
  pop   {r4-r12, lr}                             // restore registers from stack
```

Reverses the push: loads each register from the stack and increments SP.

## Addressing Modes

ARM Cortex-M33 supports several addressing modes:

| Mode | Syntax | Effective Address |
|---|---|---|
| Register | `[r0]` | r0 |
| Immediate offset | `[r0, #imm]` | r0 + imm |
| Register offset | `[r0, r1]` | r0 + r1 |
| Pre-indexed | `[r0, #imm]!` | r0 + imm (r0 updated) |
| Post-indexed | `[r0], #imm` | r0 (then r0 += imm) |

Our firmware uses only the first two: register and immediate offset.  These are the simplest and most common.

## Memory Access Sizes

| Instruction | Size | Bits | Extension |
|---|---|---|---|
| ldr / str | Word | 32 | — |
| ldrh / strh | Halfword | 16 | Zero-extend |
| ldrb / strb | Byte | 8 | Zero-extend |
| ldrsh | Signed halfword | 16 | Sign-extend |
| ldrsb | Signed byte | 8 | Sign-extend |

Our firmware uses only word-sized accesses because all RP2350 peripheral registers are 32 bits wide.

## Memory Access in Hardware Configuration

Here is a complete example from our UART initialization:

```asm
  ldr   r0, =IO_BANK0_BASE                       // r0 = 0x40028000
  ldr   r1, =2                                   // r1 = 2 (UART function)
  str   r1, [r0, #4]                             // GPIO0_CTRL = 2 (TX pin)
  str   r1, [r0, #0x0c]                          // GPIO1_CTRL = 2 (RX pin)
```

Four instructions, two stores.  The base address is loaded once, and different registers within IO_BANK0 are accessed using different offsets.

## The Polling Loop Pattern

Hardware status polling combines load and test:

```asm
.wait_loop:
  ldr   r0, =XOSC_STATUS                         // load status register address
  ldr   r1, [r0]                                 // read current status
  tst   r1, #(1<<31)                             // test STABLE bit
  beq   .wait_loop                               // loop if not stable
```

This loop executes thousands of times, reading the same address repeatedly until the hardware sets the expected bit.  Each `ldr r1, [r0]` reads a fresh value from the hardware register.

## Summary

ARM Cortex-M33 accesses memory through `ldr` (load), `str` (store), `push`, and `pop`.  Immediate offsets allow accessing multiple registers within a peripheral using a single base address.  The `ldr r0, =value` pseudo-instruction loads constants and is distinct from actual memory loads.  All peripheral register accesses in our firmware are word-sized (32-bit).  The polling loop pattern — load, test, branch — is used to wait for hardware status changes.
