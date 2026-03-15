# Chapter 1: What Is a Computer?

## Introduction

A computer is a machine that executes instructions.  That is technically all it is.  Everything else, the screen, the keyboard, the mouse, the network, the storage, those are all peripherals.  The core of a computer is a processor that reads an instruction, decodes what it means, executes it, and moves to the next instruction.  It does this billions of times per second.

This book teaches you how to program a real processor at the lowest possible level.  You will write every instruction by hand in assembly language.  You will understand every single bit and byte that makes the hardware function.  By the end of this book, you will have built a working UART echo program on the RP2350 microcontroller using ARM Cortex-M33 assembly.

## The Fetch-Decode-Execute Cycle

Every processor in existence follows this loop:

```
1. FETCH:   Read the next instruction from memory
2. DECODE:  Figure out what the instruction means
3. EXECUTE: Perform the operation
4. REPEAT:  Move to the next instruction
```

This is called the fetch-decode-execute cycle, and it is the heartbeat of every CPU.

When you write assembly, you are writing the exact instructions the CPU will fetch.  There is no compiler translating your intent.  There is no runtime interpreting your code.  The CPU reads your instruction bytes directly from memory and executes them.

## The Three Core Components

A minimal computer system consists of three things:

### 1. CPU (Central Processing Unit)

The CPU is the brain.  It contains:
- **Registers**: tiny, fast storage locations inside the processor
- **ALU** (Arithmetic Logic Unit): performs math and logic operations
- **Control Unit**: decodes instructions and sequences operations
- **Program Counter** (PC): holds the address of the next instruction

### 2. Memory

Memory holds both instructions and data.  The CPU reads instructions from memory to execute them, and reads/writes data from/to memory as the program demands.

Memory is organized as a flat array of bytes, each with a unique address.  Address 0x00000000 is the first byte, 0x00000001 is the second, and so on.

### 3. Peripherals

Peripherals are everything that is not the CPU or main memory.  On a microcontroller like the RP2350, peripherals include:
- **UART**: serial communication (what our firmware uses)
- **GPIO**: general-purpose input/output pins
- **SPI/I2C**: other communication buses
- **Timers**: counting and timing hardware
- **ADC**: analog-to-digital converter

Each peripheral is controlled by writing to and reading from specific memory addresses.  This is called memory-mapped I/O and is covered in detail in Chapter 17.

## Microcontroller vs Desktop Computer

A desktop computer has gigabytes of RAM, a hard drive, an operating system, and hundreds of running processes.  A microcontroller has kilobytes of RAM, no operating system, and runs one program from the moment it powers on until it loses power.

The RP2350 has:
- **520 KB** of SRAM (not gigabytes)
- **No operating system** — your code IS the operating system
- **No file system** — your program is stored in flash and runs directly
- **No virtual memory** — every address is a real, physical address

This is bare-metal programming.  There is nothing between your code and the hardware.

## What Is RP2350?

The RP2350 is a microcontroller made by Raspberry Pi.  It is the chip on the Raspberry Pi Pico 2 board.  Key features:

- **Dual-architecture**: can run ARM Cortex-M33 or RISC-V Hazard3 cores
- **ARM Cortex-M33**: 32-bit, Thumb-2 instruction set, 150 MHz
- **520 KB SRAM**: fast, on-chip memory
- **Up to 16 MB external flash**: connected via QSPI
- **Rich peripheral set**: UART, SPI, I2C, GPIO, PWM, ADC, timers

In this book we use the ARM Cortex-M33 core.  The processor executes Thumb-2 instructions, a compact and efficient encoding of the ARM instruction set.

## What Is ARM Cortex-M33?

ARM Cortex-M33 is a 32-bit processor core designed for embedded systems.  It belongs to the ARMv8-M architecture family.  Key characteristics:

- **Thumb-2 instruction set**: a mix of 16-bit and 32-bit instructions for code density and performance
- **32 general-purpose registers**: r0 through r15 (r13 is the stack pointer, r14 is the link register, r15 is the program counter)
- **Hardware divide**: single-cycle integer division
- **TrustZone**: hardware security extensions (not used in our firmware)
- **Nested Vectored Interrupt Controller** (NVIC): hardware interrupt management

## What Is Assembly Language?

Assembly language is a human-readable representation of machine code.  Each assembly instruction corresponds directly to one (or sometimes two) machine instructions.

```asm
  ldr   r0, =0x40070000                          // load UART0 base address into r0
```

This single line tells the processor to load the value 0x40070000 into register r0.  The assembler translates this into the exact bytes the CPU will execute.

There is a one-to-one mapping between what you write and what the CPU does.  If you write 10 instructions, the CPU executes 10 instructions.  Nothing is hidden.

## Why Learn Assembly?

1. **Complete understanding**: you know exactly what the hardware is doing
2. **No abstraction leaks**: there are no layers hiding bugs or performance problems
3. **Hardware control**: you can configure every register in every peripheral
4. **Debugging**: when something goes wrong, you can read the raw machine state
5. **Foundation**: every higher-level language compiles down to this

## What We Will Build

By the end of this book, you will have a complete, working UART echo program.  When you type a character on your computer's serial terminal, the RP2350 will receive it over UART and immediately send it back.  You will see the character appear on your screen.

This requires:
1. Configuring the crystal oscillator (clock source)
2. Enabling the peripheral clock
3. Releasing the UART from reset
4. Configuring UART pins, baud rate, and line control
5. Writing a transmit function
6. Writing a receive function
7. Writing a main loop that calls receive then transmit forever

Every single step is done in assembly.  Every single register write is explained.  Nothing is hidden behind a library or HAL.

## Summary

A computer is a processor that fetches, decodes, and executes instructions in a loop.  A microcontroller is a small computer with built-in memory and peripherals.  The RP2350 is a microcontroller with an ARM Cortex-M33 core that executes Thumb-2 instructions.  Assembly language lets you write those instructions directly.  This book teaches you every instruction, every register, and every hardware detail needed to build a working UART driver from scratch.
