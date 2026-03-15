# Chapter 11: ARM Branch Instructions

## Introduction

Branch instructions change the flow of execution.  Instead of executing the next sequential instruction, the processor jumps to a different address.  This chapter covers conditional branches, unconditional branches, and the condition flags that control them.

## Unconditional Branch: b

```asm
  b     .Loop                                    // jump to .Loop unconditionally
```

The processor sets PC to the address of `.Loop` and resumes execution there.  This is used for infinite loops (our main echo loop) and for jumping over code.

## Branch with Link: bl

```asm
  bl    UART_Init                                // call UART_Init function
```

`bl` does two things:
1. Stores the return address (address of the next instruction) in LR (r14)
2. Sets PC to the address of `UART_Init`

When `UART_Init` finishes, it executes `bx lr` to return to the instruction after the `bl`.

## Branch Exchange: bx

```asm
  bx    lr                                       // return to caller
```

`bx` branches to the address in a register.  The lowest bit selects the instruction set (1 = Thumb, which is always the case on Cortex-M).  `bx lr` is the standard function return.

## Condition Flags

Conditional branches depend on the four flags in the APSR:

| Flag | Name | Set When |
|---|---|---|
| N | Negative | Result bit 31 is 1 |
| Z | Zero | Result is zero |
| C | Carry | Unsigned overflow or carry-out |
| V | Overflow | Signed overflow |

Flags are set by instructions with the `s` suffix (`subs`, `ands`, `tst`, `cmp`).

## Conditional Branches

| Instruction | Condition | Flags | Meaning |
|---|---|---|---|
| beq | Equal / zero | Z=1 | Branch if result was zero |
| bne | Not equal | Z=0 | Branch if result was not zero |
| bgt | Greater than (signed) | Z=0, N=V | Branch if positive and non-zero |
| bge | Greater or equal (signed) | N=V | Branch if positive or zero |
| blt | Less than (signed) | N≠V | Branch if negative |
| ble | Less or equal (signed) | Z=1 or N≠V | Branch if negative or zero |
| bhi | Higher (unsigned) | C=1, Z=0 | Branch if unsigned greater |
| blo | Lower (unsigned) | C=0 | Branch if unsigned less |

## Branches Used in Our Firmware

### beq — Branch if Equal (Z=1)

```asm
  tst   r1, #(1<<31)                             // test XOSC STABLE bit
  beq   .Init_XOSC_Wait                          // loop if bit not set (Z=1)
```

`tst` computes `r1 AND (1<<31)`.  If bit 31 of r1 is 0, the result is zero, Z flag is set, and `beq` branches back to the polling loop.

```asm
  tst   r1, #(1<<6)                              // test IO_BANK0 reset done bit
  beq   .GPIO_Subsystem_Reset_Wait               // loop if not done (Z=1)
```

Same pattern: test a specific bit, loop if it is not set.

```asm
  tst   r1, #(1<<26)                             // test UART0 reset done bit
  beq   .UART_Release_Reset_Wait                 // loop if not done (Z=1)
```

### bne — Branch if Not Equal (Z=0)

```asm
  ands  r5, r5, r6                               // isolate TX FIFO full bit
  bne   .UART0_Out_loop                          // loop if FIFO is full (Z=0)
```

`ands` computes `r5 AND r6` and updates flags.  If the result is non-zero (the bit is set, FIFO is full), `bne` branches back to poll again.

```asm
  ands  r5, r5, r6                               // isolate RX FIFO empty bit
  bne   .UART0_In_loop                           // loop if FIFO is empty (Z=0)
```

Same logic: if the RX FIFO empty bit is set, keep polling.

```asm
  subs  r5, r5, #1                               // decrement delay counter
  bne   .Delay_MS_Loop                           // loop if not zero
```

Decrement and branch until counter reaches zero.  The classic countdown loop.

### ble — Branch if Less or Equal

```asm
  cmp   r0, #0                                   // compare delay value to 0
  ble   .Delay_MS_Done                           // skip if delay is 0 or negative
```

`cmp` subtracts 0 from r0 and sets flags.  If r0 ≤ 0, the delay function returns immediately without looping.

## Polling Loops

Polling (busy-waiting) is the fundamental technique in our firmware for waiting on hardware.  The pattern is always:

```asm
.wait:
  ldr   r0, =STATUS_ADDRESS                      // load status register address
  ldr   r1, [r0]                                 // read current status
  tst   r1, #BIT_MASK                            // test specific bit
  beq   .wait                                    // loop if not set
```

Three variations appear in our code:

1. **XOSC stable**: wait for bit 31 of XOSC_STATUS
2. **Reset done**: wait for bit 6 (IO_BANK0) or bit 26 (UART0) of RESETS_RESET_DONE
3. **UART FIFO**: wait for TX FIFO not full or RX FIFO not empty

## Branch Range

ARM Thumb-2 conditional branches have a range of roughly ±1 MB from the current instruction.  Unconditional `b` and `bl` have a range of roughly ±16 MB.  Our firmware is tiny (a few hundred bytes), so branch range is never a concern.

## Summary

Branch instructions control program flow.  `b` jumps unconditionally.  `bl` calls a function (saving the return address in LR).  `bx lr` returns.  Conditional branches (`beq`, `bne`, `ble`) depend on the condition flags set by previous instructions (`tst`, `cmp`, `ands`, `subs`).  The polling loop pattern — load, test, branch — is the backbone of our hardware initialization code.
