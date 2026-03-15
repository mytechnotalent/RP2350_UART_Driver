# Chapter 27: uart.s Part 1 — UART_Release_Reset and UART_Init

## Introduction

The `uart.s` file contains four functions.  This chapter covers the first two: `UART_Release_Reset` (brings UART0 out of reset) and `UART_Init` (configures pins, baud rate, and enables the peripheral).

## Function 1: UART_Release_Reset

### Full Source

```
.global UART_Release_Reset
.type UART_Release_Reset, %function
UART_Release_Reset:
  ldr   r0, =RESETS_RESET                        // load RESETS->RESET address
  ldr   r1, [r0]                                 // read RESETS->RESET value
  bic   r1, r1, #(1<<26)                         // clear UART0 reset bit
  str   r1, [r0]                                 // write value back to RESETS->RESET
.UART_Release_Reset_Wait:
  ldr   r0, =RESETS_RESET_DONE                   // load RESETS->RESET_DONE address
  ldr   r1, [r0]                                 // read RESETS->RESET_DONE value
  tst   r1, #(1<<26)                             // test UART0 reset-done bit
  beq   .UART_Release_Reset_Wait                 // loop until UART0 is out of reset
  bx    lr                                       // return
```

### Line-by-Line Walkthrough

```
  ldr   r0, =RESETS_RESET                        // load RESETS->RESET address
  ldr   r1, [r0]                                 // read RESETS->RESET value
```

Loads address 0x40020000 into r0, then reads the 32-bit reset register.

```
  bic   r1, r1, #(1<<26)                         // clear UART0 reset bit
```

Clears bit 26, which corresponds to UART0 in the reset controller.  Clearing this bit begins the process of releasing UART0 from reset.

```
  str   r1, [r0]                                 // write value back to RESETS->RESET
```

Writes the modified value back.  UART0 begins its reset release sequence.

```
.UART_Release_Reset_Wait:
  ldr   r0, =RESETS_RESET_DONE                   // load RESETS->RESET_DONE address
  ldr   r1, [r0]                                 // read RESETS->RESET_DONE value
  tst   r1, #(1<<26)                             // test UART0 reset-done bit
  beq   .UART_Release_Reset_Wait                 // loop until UART0 is out of reset
  bx    lr                                       // return
```

Polls address 0x40020008 until bit 26 is set, meaning UART0 has completed its reset release and its registers are now accessible.

This is the same reset-release pattern used in `reset.s` for IO_BANK0, just targeting a different bit.

## Function 2: UART_Init

### Full Source

```
.global UART_Init
.type UART_Init, %function
UART_Init:
  ldr   r0, =IO_BANK0_BASE                       // load IO_BANK0 base
  ldr   r1, =2                                   // FUNCSEL = 2 -> select UART function
  str   r1, [r0, #4]                             // write FUNCSEL to GPIO0_CTRL (pin0 -> TX)
  str   r1, [r0, #0x0c]                          // write FUNCSEL to GPIO1_CTRL (pin1 -> RX)
  ldr   r0, =PADS_BANK0_BASE                     // load PADS_BANK0 base
  add   r0, r0, #0x04                            // compute PAD[0] address (PADS + 0x04)
  ldr   r1, =0x04                                // pad config value for TX (pull/func recommended)
  str   r1, [r0]                                 // write PAD0 config (TX pad)
  ldr   r0, =PADS_BANK0_BASE                     // load PADS_BANK0 base again
  add   r0, r0, #0x08                            // compute PAD[1] address (PADS + 0x08)
  ldr   r1, =0x40                                // pad config value for RX (pulldown/IE as needed)
  str   r1, [r0]                                 // write PAD1 config (RX pad)
  ldr   r0, =UART0_BASE                          // load UART0 base address
  ldr   r1, =0                                   // prepare 0 to disable UARTCR
  str   r1, [r0, #0x30]                          // UARTCR = 0 (disable UART while configuring)
  ldr   r1, =6                                   // integer baud divisor (IBRD = 6)
  str   r1, [r0, #0x24]                          // UARTIBRD = 6 (integer baud divisor)
  ldr   r1, =33                                  // fractional baud divisor (FBRD = 33)
  str   r1, [r0, #0x28]                          // UARTFBRD = 33 (fractional baud divisor)
  ldr   r1, =112                                 // UARTLCR_H = 0x70 (FIFO enable + 8-bit)
  str   r1, [r0, #0x2c]                          // UARTLCR_H = 0x70 (FIFO enable + 8-bit)
  ldr   r1, =3                                   // RXE/TXE mask (will be shifted into bits 8..9)
  lsl   r1, r1, #8                               // shift RXE/TXE into bit positions 8..9
  orr   r1, r1, #1                               // set UARTEN bit (bit 0)
  str   r1, [r0, #0x30]                          // UARTCR = enable (UARTEN + TXE + RXE)
  bx    lr                                       // return
```

### Part A: GPIO Pin Configuration

```
  ldr   r0, =IO_BANK0_BASE                       // load IO_BANK0 base
  ldr   r1, =2                                   // FUNCSEL = 2 -> select UART function
  str   r1, [r0, #4]                             // write FUNCSEL to GPIO0_CTRL (pin0 -> TX)
  str   r1, [r0, #0x0c]                          // write FUNCSEL to GPIO1_CTRL (pin1 -> RX)
```

IO_BANK0_BASE is 0x40028000.  Each GPIO has a STATUS register and a CTRL register at 8-byte intervals:
- GPIO0_CTRL is at offset 0x04
- GPIO1_CTRL is at offset 0x0C

Writing FUNCSEL = 2 to the CTRL register selects the UART function for that pin:
- GPIO0 with FUNCSEL=2 becomes UART0_TX
- GPIO1 with FUNCSEL=2 becomes UART0_RX

The RP2350 multiplexes each GPIO pin among several functions (SPI, UART, I2C, PIO, SIO, etc.).  FUNCSEL selects which function is active.

### Part B: Pad Configuration

```
  ldr   r0, =PADS_BANK0_BASE                     // load PADS_BANK0 base
  add   r0, r0, #0x04                            // compute PAD[0] address (PADS + 0x04)
  ldr   r1, =0x04                                // pad config value for TX (pull/func recommended)
  str   r1, [r0]                                 // write PAD0 config (TX pad)
```

PADS_BANK0_BASE is 0x40038000.  PAD[0] is at offset 0x04.  The value 0x04 configures the TX pad:
- Bit 2 = 1 — drive strength (selected)
- Other bits = 0 — output disable cleared, input enable cleared, no pull

```
  ldr   r0, =PADS_BANK0_BASE                     // load PADS_BANK0 base again
  add   r0, r0, #0x08                            // compute PAD[1] address (PADS + 0x08)
  ldr   r1, =0x40                                // pad config value for RX (pulldown/IE as needed)
  str   r1, [r0]                                 // write PAD1 config (RX pad)
```

PAD[1] is at offset 0x08.  The value 0x40 configures the RX pad:
- Bit 6 = 1 — input enable (IE), required for the pad to receive data
- Other bits = 0 — no pull, output disable cleared

### Part C: Disable UART During Configuration

```
  ldr   r0, =UART0_BASE                          // load UART0 base address
  ldr   r1, =0                                   // prepare 0 to disable UARTCR
  str   r1, [r0, #0x30]                          // UARTCR = 0 (disable UART while configuring)
```

UART0_BASE is 0x40070000.  UARTCR (Control Register) is at offset 0x30.  Writing 0 disables the UART completely.  The PL011 datasheet recommends disabling the UART before changing baud rate or line control settings.

### Part D: Baud Rate Configuration

```
  ldr   r1, =6                                   // integer baud divisor (IBRD = 6)
  str   r1, [r0, #0x24]                          // UARTIBRD = 6 (integer baud divisor)
  ldr   r1, =33                                  // fractional baud divisor (FBRD = 33)
  str   r1, [r0, #0x28]                          // UARTFBRD = 33 (fractional baud divisor)
```

UARTIBRD at offset 0x24 holds the integer part of the baud-rate divisor.  UARTFBRD at offset 0x28 holds the fractional part.

The baud rate formula for PL011:

```
  Baud Rate = UARTCLK / (16 x Divisor)
  Divisor   = IBRD + (FBRD / 64)
  Divisor   = 6 + (33 / 64) = 6.515625
  Baud Rate = 12,000,000 / (16 x 6.515625)
  Baud Rate = 12,000,000 / 104.25
  Baud Rate = 115,107.9 (approximately 115,200)
```

The error is less than 0.1%, well within the 3% UART tolerance.

### Part E: Line Control

```
  ldr   r1, =112                                 // UARTLCR_H = 0x70 (FIFO enable + 8-bit)
  str   r1, [r0, #0x2c]                          // UARTLCR_H = 0x70 (FIFO enable + 8-bit)
```

UARTLCR_H at offset 0x2C controls the data format.  The value 112 = 0x70:

```
  Bit 4 (FEN)  = 1  -> FIFO enable
  Bit 5 (WLEN0)= 1  -> |
  Bit 6 (WLEN1)= 1  -> | Word length = 8 bits (11 = 8 bits)
```

This configures: 8 data bits, no parity, 1 stop bit, FIFOs enabled.  The FIFOs are 16 entries deep and reduce the number of interrupts (or poll iterations) needed.

### Part F: Enable UART

```
  ldr   r1, =3                                   // RXE/TXE mask (will be shifted into bits 8..9)
  lsl   r1, r1, #8                               // shift RXE/TXE into bit positions 8..9
  orr   r1, r1, #1                               // set UARTEN bit (bit 0)
  str   r1, [r0, #0x30]                          // UARTCR = enable (UARTEN + TXE + RXE)
  bx    lr                                       // return
```

Builds the UARTCR enable value:
1. `ldr r1, =3` — loads 0x00000003 (binary: 11)
2. `lsl r1, r1, #8` — shifts left 8 positions: 0x00000300.  This sets bit 8 (TXE = transmit enable) and bit 9 (RXE = receive enable)
3. `orr r1, r1, #1` — sets bit 0 (UARTEN = UART enable): 0x00000301

The final value 0x301 enables the UART with both transmit and receive.

## UART Register Map Summary

```
  Offset  Register    Value     Purpose
  ------  --------    -----     -------
  0x00    UARTDR      (data)    Data register (read/write)
  0x18    UARTFR      (status)  Flag register (TXFF, RXFE)
  0x24    UARTIBRD    6         Integer baud divisor
  0x28    UARTFBRD    33        Fractional baud divisor
  0x2C    UARTLCR_H   0x70     Line control (8N1, FIFO)
  0x30    UARTCR      0x301    Control (enable TX, RX, UART)
```

## Summary

`UART_Release_Reset` follows the same reset-release pattern as `Init_Subsystem`: clear the reset bit for UART0 (bit 26) and poll the done bit.  `UART_Init` configures the UART in six steps: set GPIO pin functions, configure pads, disable UART, set baud rate divisors, set line control, and re-enable with TX/RX active.  After these two functions complete, UART0 is ready to send and receive data.
