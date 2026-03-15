# Chapter 28: uart.s Part 2 — UART0_Out and UART0_In

## Introduction

With UART0 initialized, the firmware needs two functions: one to send a byte and one to receive a byte.  Both use polling — they loop until the UART hardware is ready, then transfer one byte.  This chapter explains every instruction in `UART0_Out` and `UART0_In`.

## Function 1: UART0_Out (Transmit)

### Full Source

```
.global UART0_Out
.type UART0_Out, %function
UART0_Out:
.UART0_Out_Push_Registers:
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
.UART0_Out_loop:
  ldr   r4, =UART0_BASE                          // base address for uart0 registers
  ldr   r5, [r4, #0x18]                          // read UART0 flag register UARTFR into r5
  ldr   r6, =32                                  // mask for bit 5, TX FIFO full (TXFF)
  ands  r5, r5, r6                               // isolate TXFF bit and set flags
  bne   .UART0_Out_loop                          // if TX FIFO is full, loop
  ldr   r6, =0xff                                // mask for the 8 lowest bits
  ands  r0, r0, r6                               // mask off upper bits of r0, keep lower 8 bits
  str   r0, [r4, #0]                             // write data to UARTDR
.UART0_Out_Pop_Registers:
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return
```

### Line-by-Line Walkthrough

#### Save Registers

```
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
```

Saves callee-saved registers (r4-r12) and the link register to the stack.  This function uses r4, r5, and r6 for scratch work, so they must be saved and restored.  The caller's data to transmit is in r0, which is caller-saved and does not need to be pushed.

#### Poll the TX FIFO

```
.UART0_Out_loop:
  ldr   r4, =UART0_BASE                          // base address for uart0 registers
  ldr   r5, [r4, #0x18]                          // read UART0 flag register UARTFR into r5
  ldr   r6, =32                                  // mask for bit 5, TX FIFO full (TXFF)
  ands  r5, r5, r6                               // isolate TXFF bit and set flags
  bne   .UART0_Out_loop                          // if TX FIFO is full, loop
```

UARTFR (Flag Register) is at offset 0x18 from UART0_BASE (0x40070018).  The relevant bits:

```
  Bit   Name   Meaning
  ----  ----   -------
  3     BUSY   UART is transmitting
  4     RXFE   RX FIFO empty
  5     TXFF   TX FIFO full
  7     TXFE   TX FIFO empty
```

The polling loop:
1. `ldr r4, =UART0_BASE` — loads 0x40070000 into r4 from the literal pool
2. `ldr r5, [r4, #0x18]` — reads UARTFR into r5
3. `ldr r6, =32` — loads the mask for bit 5 (0x20 = 32 decimal)
4. `ands r5, r5, r6` — ANDs r5 with the mask and sets flags.  If bit 5 is set (FIFO full), the result is nonzero
5. `bne .UART0_Out_loop` — if nonzero (FIFO full), branch back and try again

The loop repeats until the TX FIFO has at least one free slot.

#### Mask and Write

```
  ldr   r6, =0xff                                // mask for the 8 lowest bits
  ands  r0, r0, r6                               // mask off upper bits of r0, keep lower 8 bits
  str   r0, [r4, #0]                             // write data to UARTDR
```

UARTDR (Data Register) is at offset 0x00.  Only the lower 8 bits are used for transmit data.  The `ands r0, r0, r6` masks r0 to ensure only valid data bits are written.  The `str` writes the byte to the UART TX FIFO.

#### Restore and Return

```
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return
```

Restores all saved registers and returns.  The byte has been placed in the TX FIFO; the UART hardware shifts it out on the TX wire asynchronously.

## Function 2: UART0_In (Receive)

### Full Source

```
.global UART0_In
.type UART0_In, %function
UART0_In:
.UART0_In_Push_Registers:
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
.UART0_In_loop:
  ldr   r4, =UART0_BASE                          // base address for uart0 registers
  ldr   r5, [r4, #0x18]                          // read UART0 flag register UARTFR into r5
  ldr   r6, =16                                  // mask for bit 4, RX FIFO empty RXFE
  ands  r5, r5, r6                               // isolate RXFE bit and set flags
  bne   .UART0_In_loop                           // if RX FIFO is empty, loop
  ldr   r0, [r4, #0]                             // load data from UARTDR into r0 (return value)
.UART0_In_Pop_Registers:
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return
```

### Line-by-Line Walkthrough

#### Save Registers

```
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
```

Same register save as UART0_Out.  Uses r4, r5, r6 for scratch work.

#### Poll the RX FIFO

```
.UART0_In_loop:
  ldr   r4, =UART0_BASE                          // base address for uart0 registers
  ldr   r5, [r4, #0x18]                          // read UART0 flag register UARTFR into r5
  ldr   r6, =16                                  // mask for bit 4, RX FIFO empty RXFE
  ands  r5, r5, r6                               // isolate RXFE bit and set flags
  bne   .UART0_In_loop                           // if RX FIFO is empty, loop
```

This time we check bit 4 (RXFE = RX FIFO Empty).  The mask is 16 = 0x10:
1. Read UARTFR into r5
2. AND with 16 to isolate bit 4
3. If nonzero (FIFO empty), loop — there is nothing to read
4. If zero (FIFO not empty), at least one byte is available

The polling logic is inverted compared to UART0_Out:
- UART0_Out loops while bit is SET (FIFO full → wait)
- UART0_In loops while bit is SET (FIFO empty → wait)

#### Read Data

```
  ldr   r0, [r4, #0]                             // load data from UARTDR into r0 (return value)
```

Reads UARTDR (offset 0x00).  On read, UARTDR returns the oldest byte in the RX FIFO.  The byte is placed in r0, which is the return register per AAPCS.  The upper bits (11:8) contain error flags (overrun, break, parity, framing), but our firmware ignores them.

#### Restore and Return

```
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return
```

Returns with the received byte in r0.

## Register Usage Comparison

```
  Register  UART0_Out              UART0_In
  --------  ---------              --------
  r0        input byte to send     output received byte
  r4        UART0_BASE             UART0_BASE
  r5        UARTFR value           UARTFR value
  r6        bit mask (32 or 0xFF)  bit mask (16)
```

## The Polling Pattern

Both functions follow the same polling pattern:

```
  1. Load the flag register
  2. Mask the relevant bit
  3. Branch if not ready
  4. Perform the data transfer
```

This is busy-waiting — the CPU does nothing useful while polling.  In a more complex system, you would use interrupts instead.  For a simple echo loop, polling is sufficient and simpler to implement.

## Data Flow Through the UART

```
  Host computer         RP2350
  sends byte   ------> RX wire ------> UART0 RX FIFO
                                              |
                                        UART0_In reads UARTDR
                                              |
                                        r0 = received byte
                                              |
                                        UART0_Out writes UARTDR
                                              |
                                        UART0 TX FIFO ------> TX wire ------> Host computer
                                                                               receives byte
```

## Summary

`UART0_Out` polls bit 5 (TXFF) of the flag register until the TX FIFO has space, masks the byte to 8 bits, and writes it to UARTDR.  `UART0_In` polls bit 4 (RXFE) until the RX FIFO has data, then reads UARTDR into r0.  Both functions save and restore callee-saved registers.  Together, they provide blocking single-byte send and receive for the echo loop in `main`.
