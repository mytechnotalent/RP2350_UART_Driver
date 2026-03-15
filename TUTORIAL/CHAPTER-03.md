# Chapter 3: Memory — Addresses, Bytes, Words, and Endianness

## Introduction

Memory is a flat array of bytes.  Every byte has a unique address.  The processor reads and writes memory using these addresses.  Understanding how memory is organized, how multi-byte values are stored, and how the address space is divided is essential before writing any code.

## The Address Space

ARM Cortex-M33 uses a 32-bit address space.  That means addresses range from 0x00000000 to 0xFFFFFFFF, giving 4 GB of addressable space.  On the RP2350, most of this space is unused.  Only specific ranges are mapped to real hardware:

```
  0x00000000 - 0x0FFFFFFF    ROM (bootloader)
  0x10000000 - 0x11FFFFFF    External Flash (XIP)
  0x20000000 - 0x20081FFF    SRAM (520 KB)
  0x40000000 - 0x4FFFFFFF    Peripheral registers (APB)
  0xE0000000 - 0xE00FFFFF    Private Peripheral Bus (PPB)
```

Our firmware lives in flash starting at 0x10000000.  The stack lives in SRAM starting at the top of 0x20082000.  Peripheral registers like UART0 live at 0x40070000.

## Bytes, Halfwords, and Words

| Unit | Size | Bits |
|---|---|---|
| Byte | 1 byte | 8 bits |
| Halfword | 2 bytes | 16 bits |
| Word | 4 bytes | 32 bits |

ARM Cortex-M33 is a 32-bit architecture.  Its registers are 32 bits (one word) wide.  Most load and store instructions operate on full words.

## Alignment

ARM Cortex-M33 requires aligned memory accesses by default:
- **Word access** (ldr/str): address must be divisible by 4
- **Halfword access** (ldrh/strh): address must be divisible by 2
- **Byte access** (ldrb/strb): any address is valid

If you attempt a word load from an address that is not word-aligned, the processor generates a HardFault exception.  In our firmware, all register addresses are word-aligned (ending in 0, 4, 8, or C in hex).

## Little-Endian Byte Order

ARM Cortex-M33 on the RP2350 uses little-endian byte order.  This means the least significant byte is stored at the lowest address.

```
  Word value: 0x12345678
  Stored at address 0x20000000:

  Address:      0x20000000  0x20000001  0x20000002  0x20000003
  Byte value:   0x78        0x56        0x34        0x12
                (LSB)                               (MSB)
```

When the processor executes `ldr r0, [r1]` where r1 = 0x20000000, it reads all four bytes and assembles them back into 0x12345678 in r0.

You rarely need to think about endianness in normal code.  It matters when you are looking at raw memory dumps or when communicating with big-endian systems.

## Memory-Mapped Registers

On the RP2350, hardware peripherals are controlled by reading and writing specific memory addresses.  These addresses are called memory-mapped registers.

```asm
  ldr   r0, =0x40070000                          // r0 = UART0 base address
  ldr   r1, [r0, #0x18]                          // r1 = value at UART0 + 0x18 (UARTFR)
```

Address 0x40070018 is not RAM — it is a hardware register.  Reading from it returns the current state of the UART flag register.  Writing to other offsets configures the UART.  Chapter 17 covers memory-mapped I/O in full detail.

## The Stack

The stack is a region of SRAM used for temporary storage.  It grows downward — from high addresses to low addresses.  When you `push` a register, the stack pointer decreases.  When you `pop`, it increases.

```
  Stack Top:    0x20082000  (highest address, nothing stored here)
  After push:   0x20081FFC  (4 bytes used, one word stored)
  After push:   0x20081FF8  (8 bytes used, two words stored)
  Stack Limit:  0x2007A000  (lowest allowed address)
```

Our firmware initializes the stack pointer to STACK_TOP (0x20082000) and sets the stack limit to STACK_LIMIT (0x2007A000).  This gives 32 KB of stack space.

## Flash Memory (XIP)

Our program is stored in external flash starting at address 0x10000000.  The RP2350 uses Execute-in-Place (XIP) — the processor executes instructions directly from flash through a cache, without copying them to SRAM first.

The flash contents are:
1. IMAGE_DEF block (boot metadata) — must be in first 4 KB
2. Vector table (stack pointer and reset handler address)
3. Program code (.text section)
4. Read-only data (.rodata section)

## SRAM

The RP2350 has 520 KB of SRAM at 0x20000000.  Our firmware uses a small portion for the stack.  We do not use heap allocation.  We do not use the .data or .bss sections (no global variables).

## Reading the Address Map

When you see an address like 0x40070024 in our firmware, you can decode it:

```
  0x40070024
  ├── 0x40000000  = APB peripheral space
  ├── 0x00070000  = UART0 peripheral offset
  └── 0x00000024  = UARTIBRD register offset within UART0
```

Every address in our code points to a real, physical location — either flash, SRAM, or a peripheral register.  There is no virtual memory translation.  What you write is what the hardware sees.

## Summary

Memory is a flat array of bytes with 32-bit addresses.  ARM Cortex-M33 stores multi-byte values in little-endian order.  Word accesses must be 4-byte aligned.  Flash holds the program code at 0x10000000.  SRAM holds the stack at the top of 0x20082000.  Peripheral registers at 0x40000000 and above control hardware.  Every address in our firmware corresponds to a real hardware location.
