# Chapter 30: Full Integration — From Power-On to Echo

## Introduction

This final chapter ties every piece together.  We trace the complete journey from the moment power reaches the RP2350 to the moment a character echoes back to the host terminal.  Every file, every function, every register write — all connected in sequence.

## The Files

Our firmware consists of 12 source files, 1 linker script, and 1 build script:

```
  File               Role
  ----               ----
  image_def.s        PICOBIN metadata block
  vector_table.s     Initial SP and reset vector
  constants.s        All hardware addresses
  stack.s            Stack pointer initialization
  reset_handler.s    Boot orchestrator
  xosc.s             Crystal oscillator setup
  reset.s            GPIO subsystem reset release
  uart.s             UART reset, init, TX, RX
  coprocessor.s      Coprocessor access enable
  gpio.s             GPIO config, set, clear
  delay.s            Millisecond delay
  main.s             Echo loop
  linker.ld          Memory layout and section placement
  build.bat          Build pipeline
```

## Phase 1: Build

The build script transforms assembly into a flashable binary:

```
  12 .s files --> arm-none-eabi-as --> 12 .o files
  12 .o files + linker.ld --> arm-none-eabi-ld --> uart.elf
  uart.elf --> arm-none-eabi-objcopy --> uart.bin
  uart.bin --> uf2conv.py --> uart.uf2
```

The assembler encodes Thumb-2 instructions.  The linker places the IMAGE_DEF block first, the vector table at a 128-byte boundary, and all code in flash.  objcopy strips ELF metadata.  uf2conv.py wraps the binary with address information and the RP2350 ARM-S family ID (0xe48bff59).

## Phase 2: Flash

The UF2 file is loaded onto the RP2350.  Each 512-byte UF2 block carries up to 256 bytes of payload and a target address starting at 0x10000000.  The boot ROM writes each payload to its designated flash address.

## Phase 3: Boot ROM

When the RP2350 powers on:

1. The internal boot ROM executes from ROM
2. It scans the first 4 KB of flash for a valid PICOBIN block
3. It finds our IMAGE_DEF at address 0x10000000:
   - Marker start: 0xFFFFDED3
   - Image type: ARM Secure EXE (0x1021)
   - Marker end: 0xAB123579
4. The boot ROM validates the block and prepares to execute our binary
5. The CPU reads the vector table at 0x10000080

## Phase 4: Hardware Reset Sequence

The CPU reads two words from the vector table:

```
  Address      Value         Action
  ----------   ----------    ------
  0x10000080   0x20082000    Load into MSP (initial stack pointer)
  0x10000084   Reset_Handler + 1   Load into PC (reset vector, Thumb bit set)
```

The CPU clears bit 0 of the reset vector to get the actual address, enters Thumb mode, and begins executing at Reset_Handler.

## Phase 5: Reset_Handler

Reset_Handler calls seven functions in order:

### Step 1: Init_Stack (stack.s)

```
  ldr   r0, =STACK_TOP                           // load 0x20082000
  msr   PSP, r0                                  // set Process Stack Pointer
  ldr   r0, =STACK_LIMIT                         // load 0x2007A000
  msr   MSPLIM, r0                               // set MSP lower limit
  msr   PSPLIM, r0                               // set PSP lower limit
  ldr   r0, =STACK_TOP                           // reload 0x20082000
  msr   MSP, r0                                  // set Main Stack Pointer
```

Result: All four stack registers configured with hardware overflow protection.

### Step 2: Init_XOSC (xosc.s)

```
  XOSC_STARTUP = 0x00C4                          // 50,000-cycle startup delay
  XOSC_CTRL    = 0x00FABAA0                       // enable, 1-15MHz range
  Poll XOSC_STATUS bit 31 until STABLE
```

Result: External 12 MHz crystal oscillator running and stable.

### Step 3: Enable_XOSC_Peri_Clock (xosc.s)

```
  CLK_PERI_CTRL |= (1<<11)                       // enable peripheral clock
  CLK_PERI_CTRL |= (4<<5)                        // select XOSC as source
```

Result: Peripheral clock domain running from the stable crystal at 12 MHz.

### Step 4: Init_Subsystem (reset.s)

```
  RESETS_RESET &= ~(1<<6)                        // release IO_BANK0
  Poll RESETS_RESET_DONE bit 6 until done
```

Result: IO_BANK0 (GPIO function select) is out of reset and accessible.

### Step 5: UART_Release_Reset (uart.s)

```
  RESETS_RESET &= ~(1<<26)                       // release UART0
  Poll RESETS_RESET_DONE bit 26 until done
```

Result: UART0 peripheral is out of reset and its registers are accessible.

### Step 6: UART_Init (uart.s)

```
  GPIO0_CTRL = 2                                 // pin 0 = UART TX
  GPIO1_CTRL = 2                                 // pin 1 = UART RX
  PAD0 = 0x04                                    // TX pad config
  PAD1 = 0x40                                    // RX pad config (IE)
  UARTCR = 0                                     // disable while configuring
  UARTIBRD = 6                                   // integer baud divisor
  UARTFBRD = 33                                  // fractional baud divisor
  UARTLCR_H = 0x70                               // 8N1, FIFO enable
  UARTCR = 0x301                                 // enable UART + TX + RX
```

Result: UART0 configured for 115200 baud, 8 data bits, no parity, 1 stop bit, FIFOs enabled.

### Step 7: Enable_Coprocessor (coprocessor.s)

```
  CPACR |= (1<<1) | (1<<0)                       // enable CP0 access
  dsb                                            // data sync barrier
  isb                                            // instruction sync barrier
```

Result: Coprocessor 0 accessible for GPIO coprocessor instructions.

### Branch to Main

```
  b     main                                     // one-way branch to echo loop
```

## Phase 6: The Echo Loop (main.s)

```
main:
  push  {r4-r12, lr}                             // save registers (once)
.Loop:
  bl    UART0_In                                 // block until byte received -> r0
  bl    UART0_Out                                // send byte in r0 -> TX wire
  b     .Loop                                    // repeat forever
```

### One Echo Cycle

1. `UART0_In` polls UARTFR bit 4 (RXFE) in a tight loop
2. A byte arrives on the RX wire and enters the RX FIFO
3. RXFE clears (FIFO not empty)
4. `UART0_In` reads UARTDR: the byte moves from the FIFO to r0
5. `UART0_In` returns to `main`
6. `main` calls `UART0_Out` with the byte still in r0
7. `UART0_Out` polls UARTFR bit 5 (TXFF) — typically zero immediately
8. `UART0_Out` masks r0 to 8 bits and writes to UARTDR
9. The byte enters the TX FIFO and is shifted out on the TX wire
10. The host terminal receives and displays the character
11. `b .Loop` — back to waiting for the next byte

## The Complete Address Map

```
  Address Range          Contents
  ----------------       --------
  0x10000000-0x10000013  IMAGE_DEF block (20 bytes)
  0x10000080-0x10000087  Vector table (8 bytes)
  0x10000088-0x100xxxxx  .text section (all code)

  0x2007A000             Stack limit (MSPLIM/PSPLIM)
  0x20082000             Stack top (MSP/PSP initial)

  0x40010048             CLK_PERI_CTRL
  0x40020000             RESETS_RESET
  0x40020008             RESETS_RESET_DONE
  0x40028000             IO_BANK0_BASE
  0x40038000             PADS_BANK0_BASE
  0x40048000             XOSC_BASE
  0x40070000             UART0_BASE
  0xE000ED88             CPACR
```

## What We Built

A complete bare-metal UART echo driver for the RP2350 Cortex-M33:

- **12 source files** — each with a single responsibility
- **1 linker script** — placing code exactly where the hardware expects it
- **1 build script** — four-stage pipeline from assembly to UF2
- **7 initialization steps** — executed in dependency order
- **3-instruction main loop** — read, write, repeat
- **0 dependencies** — no SDK, no C runtime, no libraries

Every byte in flash has a purpose.  Every register write follows from the RP2350 datasheet.  Every function exists because the one after it depends on it.  This is bare-metal programming: direct conversation between software and silicon.
