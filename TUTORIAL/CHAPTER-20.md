# Chapter 20: The Build Pipeline — From Assembly to UF2

## Introduction

Building bare-metal firmware requires multiple tools working in sequence.  Our build pipeline has four stages: assemble, link, extract binary, convert to UF2.  This chapter explains every step in the pipeline by walking through the build script.

## Our Build Script: build.bat

```
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb image_def.s -o image_def.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb vector_table.s -o vector_table.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb reset_handler.s -o reset_handler.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb constants.s -o constants.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb stack.s -o stack.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb xosc.s -o xosc.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb reset.s -o reset.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb uart.s -o uart.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb coprocessor.s -o coprocessor.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb gpio.s -o gpio.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb delay.s -o delay.o
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb main.s -o main.o
arm-none-eabi-ld -T linker.ld image_def.o vector_table.o reset_handler.o constants.o stack.o xosc.o reset.o uart.o coprocessor.o gpio.o delay.o main.o -o uart.elf
arm-none-eabi-objcopy -O binary uart.elf uart.bin
python uf2conv.py uart.bin --base 0x10000000 --family 0xe48bff59 --output uart.uf2
```

## Stage 1: Assembly

```
arm-none-eabi-as -g -mcpu=cortex-m33 -mthumb image_def.s -o image_def.o
```

Each `.s` file is assembled into an ELF object file (`.o`).  The flags:

- **arm-none-eabi-as** — the GNU assembler for ARM bare-metal targets.  `arm` = ARM architecture.  `none` = no operating system.  `eabi` = Embedded Application Binary Interface
- **-g** — include debug symbols in the object file, needed for GDB debugging
- **-mcpu=cortex-m33** — target the Cortex-M33 processor, enabling all instructions it supports
- **-mthumb** — generate Thumb-2 code.  Cortex-M33 only executes Thumb-2 instructions

This step runs 12 times, once per source file.  The order does not matter at this stage — the assembler processes each file independently.

### What the Assembler Actually Does

1. Reads the `.s` file
2. Parses assembly directives (.syntax, .cpu, .section, etc.)
3. Encodes each instruction into Thumb-2 machine code
4. Creates an ELF object file containing machine code, symbols, and relocation entries
5. Symbols not defined in this file are marked as "undefined" for the linker to resolve

## Stage 2: Linking

```
arm-none-eabi-ld -T linker.ld image_def.o vector_table.o ... main.o -o uart.elf
```

The linker combines all 12 object files into one ELF executable:

- **arm-none-eabi-ld** — the GNU linker for ARM bare-metal
- **-T linker.ld** — use our linker script to control section placement
- **image_def.o ... main.o** — the 12 object files in order
- **-o uart.elf** — output a single ELF executable

### What the Linker Actually Does

1. Reads each object file
2. Merges sections: all `.text` into one `.text`, all `.embedded_block` into one block
3. Resolves symbols — `bl UART_Init` in `reset_handler.o` is linked to the `UART_Init` symbol in `uart.o`
4. Applies the linker script to assign physical addresses
5. Checks the ASSERT conditions
6. Produces the final ELF with absolute addresses

### Object File Order and the Linker Script

The order of object files on the command line matters.  The linker processes sections in the order they appear.  `image_def.o` is listed first so its `.embedded_block` section appears first in flash.  `vector_table.o` is next so the vector table follows immediately.

## Stage 3: Binary Extraction

```
arm-none-eabi-objcopy -O binary uart.elf uart.bin
```

The ELF file contains metadata (headers, symbol tables, debug info) that the hardware does not need.  `objcopy` strips all metadata and produces a flat binary — just the raw bytes that will be written to flash.

- **-O binary** — output format is raw binary
- **uart.elf** — input ELF
- **uart.bin** — output flat binary

The resulting `uart.bin` starts at offset 0.  When loaded at flash address 0x10000000, byte 0 of the file maps to address 0x10000000.

## Stage 4: UF2 Conversion

```
python uf2conv.py uart.bin --base 0x10000000 --family 0xe48bff59 --output uart.uf2
```

UF2 (USB Flashing Format) is a file format designed by Microsoft for flashing microcontrollers over USB mass storage.  The Pico appears as a USB drive in BOOTSEL mode, and you simply copy the `.uf2` file to it.

- **uf2conv.py** — Python script that converts a flat binary to UF2 format
- **--base 0x10000000** — the target address in flash where the binary should be loaded
- **--family 0xe48bff59** — the RP2350 ARM-S (secure) family ID.  The boot ROM checks this to ensure the file is for the correct chip
- **--output uart.uf2** — the output UF2 file

### UF2 File Structure

A UF2 file is a series of 512-byte blocks.  Each block contains:

```
  Offset  Size  Description
  ------  ----  -----------
  0x00    4     Magic number 1: 0x0A324655
  0x04    4     Magic number 2: 0x9E5D5157
  0x08    4     Flags
  0x0C    4     Target address for this block
  0x10    4     Data payload size (up to 256 bytes)
  0x14    4     Block sequence number
  0x18    4     Total number of blocks
  0x1C    4     Family ID (0xe48bff59 for RP2350 ARM-S)
  0x20    256   Data payload
  0x120   4     Magic number 3: 0x0AB16F30
```

Each block carries up to 256 bytes of payload and its target address.  The boot ROM reads these blocks and writes the payload to the specified flash address.

## The Complete Flow

```
  image_def.s ----+
  vector_table.s -+
  reset_handler.s +
  constants.s ----+
  stack.s --------+---> arm-none-eabi-as ---> 12 .o files
  xosc.s --------+
  reset.s --------+
  uart.s ---------+
  coprocessor.s --+
  gpio.s ---------+
  delay.s --------+
  main.s ---------+

  12 .o files + linker.ld ---> arm-none-eabi-ld ---> uart.elf

  uart.elf ---> arm-none-eabi-objcopy ---> uart.bin

  uart.bin ---> uf2conv.py ---> uart.uf2
```

## Cleaning

The `clean.bat` script removes all generated files:

```
del *.o *.elf *.bin
```

This deletes all object files, the ELF, and the flat binary.  Note that `uart.uf2` is not deleted — it is kept as the deployable artifact.

## The Family ID: 0xe48bff59

The RP2350 supports multiple family IDs depending on the boot mode:

- **0xe48bff59** — ARM Secure (our firmware)
- **0xe48bff57** — ARM Non-Secure
- **0xe48bff58** — RISC-V

Using the wrong family ID means the boot ROM will reject the UF2 file.  Our firmware boots in ARM Secure mode, so we use 0xe48bff59.

## Summary

The build pipeline transforms human-readable assembly into a flash-ready UF2 file in four stages.  The assembler encodes instructions.  The linker places sections at physical addresses.  objcopy strips metadata.  uf2conv.py wraps the binary in the UF2 format.  Each stage is essential.
