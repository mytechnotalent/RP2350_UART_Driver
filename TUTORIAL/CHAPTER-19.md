# Chapter 19: The Linker Script — Placing Code in Memory

## Introduction

The assembler converts each source file into an object file.  The linker combines all object files into a single binary.  The linker script tells the linker where to place each section in the final binary — which addresses in flash, which in SRAM.  Without a correct linker script, the binary will not boot.

## Our Linker Script: linker.ld

Here is the complete linker script, followed by a line-by-line explanation:

```
ENTRY(Reset_Handler)

__XIP_BASE   = 0x10000000;
__XIP_SIZE   = 32M;

__SRAM_BASE  = 0x20000000;
__SRAM_SIZE  = 512K;
__STACK_SIZE = 32K;

MEMORY
{
  RAM   (rwx) : ORIGIN = __SRAM_BASE, LENGTH = __SRAM_SIZE
  FLASH (rx)  : ORIGIN = __XIP_BASE,  LENGTH = __XIP_SIZE
}

PHDRS
{
  text PT_LOAD FLAGS(5);
}

SECTIONS
{
  . = ORIGIN(FLASH);

  .embedded_block :
  {
    KEEP(*(.embedded_block))
  } > FLASH :text

  .vectors ALIGN(128) :
  {
    KEEP(*(.vectors))
  } > FLASH :text

  ASSERT(((ADDR(.vectors) - ORIGIN(FLASH)) < 0x1000),
         "Vector table must be in first 4KB of flash")

  .text :
  {
    . = ALIGN(4);
    *(.text*)
    *(.rodata*)
    KEEP(*(.ARM.attributes))
  } > FLASH :text

  __StackTop   = ORIGIN(RAM) + LENGTH(RAM);
  __StackLimit = __StackTop - __STACK_SIZE;
  __stack      = __StackTop;

  .stack (NOLOAD) : { . = ALIGN(8); } > RAM

  PROVIDE(__Vectors = ADDR(.vectors));
}
```

## Line-by-Line Explanation

### ENTRY(Reset_Handler)

Declares `Reset_Handler` as the program entry point.  The debugger uses this to know where execution begins.  The hardware actually uses the vector table, but ENTRY helps tools and linker optimizations.

### Memory Constants

```
__XIP_BASE   = 0x10000000;
__XIP_SIZE   = 32M;
__SRAM_BASE  = 0x20000000;
__SRAM_SIZE  = 512K;
__STACK_SIZE = 32K;
```

These define the physical memory layout of the RP2350:
- Flash starts at 0x10000000 and can be up to 32 MB
- SRAM starts at 0x20000000 and is 512 KB (the non-secure window)
- Stack is 32 KB

### MEMORY Block

```
MEMORY
{
  RAM   (rwx) : ORIGIN = __SRAM_BASE, LENGTH = __SRAM_SIZE
  FLASH (rx)  : ORIGIN = __XIP_BASE,  LENGTH = __XIP_SIZE
}
```

Defines two memory regions:
- **RAM**: read-write-execute, starting at 0x20000000
- **FLASH**: read-execute, starting at 0x10000000

The linker uses these regions to place sections.

### PHDRS Block

```
PHDRS
{
  text PT_LOAD FLAGS(5);
}
```

Defines a single program header segment of type PT_LOAD with flags 5 (read + execute).  This tells the ELF loader that the segment should be loaded into memory.

### Section Placement

#### .embedded_block (IMAGE_DEF)

```
.embedded_block :
{
  KEEP(*(.embedded_block))
} > FLASH :text
```

Places the PICOBIN block first in flash.  `KEEP` prevents the linker from discarding this section even though no code references it.  The boot ROM expects this metadata in the first 4 KB of flash.

**Note**: The section in our `image_def.s` is actually named `.picobin_block`.  The linker script uses `.embedded_block`.  If these names do not match, the IMAGE_DEF will not be placed correctly.  Both names must agree.

#### .vectors (Vector Table)

```
.vectors ALIGN(128) :
{
  KEEP(*(.vectors))
} > FLASH :text
```

Places the vector table on a 128-byte boundary in flash.  ARM Cortex-M33 requires the vector table to be naturally aligned.  `KEEP` ensures it is not discarded.

#### ASSERT

```
ASSERT(((ADDR(.vectors) - ORIGIN(FLASH)) < 0x1000),
       "Vector table must be in first 4KB of flash")
```

Compile-time check: the vector table must be within the first 4 KB of flash (offset < 0x1000).  If the IMAGE_DEF block is too large, this assertion fails and the build stops.

#### .text (Code)

```
.text :
{
  . = ALIGN(4);
  *(.text*)
  *(.rodata*)
  KEEP(*(.ARM.attributes))
} > FLASH :text
```

All code (.text) and read-only data (.rodata) from all object files are placed in flash.  `.ARM.attributes` records the ARM architecture version for tools.

### Stack Symbols

```
__StackTop   = ORIGIN(RAM) + LENGTH(RAM);
__StackLimit = __StackTop - __STACK_SIZE;
__stack      = __StackTop;
```

These create linker symbols:
- `__StackTop` = 0x20000000 + 512K = 0x20080000
- `__StackLimit` = 0x20080000 - 32K = 0x20078000
- `__stack` = same as __StackTop

Our firmware does not use these linker symbols directly — it defines STACK_TOP and STACK_LIMIT as `.equ` constants in `constants.s`.

### PROVIDE

```
PROVIDE(__Vectors = ADDR(.vectors));
```

Creates the `__Vectors` symbol pointing to the vector table address, but only if nothing else defines it.

## The Memory Layout

After linking, the flash layout looks like:

```
  0x10000000: IMAGE_DEF block (picobin metadata)
  0x10000080: Vector table (128-byte aligned)
              - .word STACK_TOP
              - .word Reset_Handler + 1
  0x10000088: .text section (all code)
              - Reset_Handler
              - Init_Stack
              - Init_XOSC
              - Enable_XOSC_Peri_Clock
              - Init_Subsystem
              - UART_Release_Reset
              - UART_Init
              - Enable_Coprocessor
              - UART0_Out
              - UART0_In
              - main
              - Delay_MS
              - GPIO_Config, GPIO_Set, GPIO_Clear
              - (literal pools)
```

## Summary

The linker script controls the binary layout.  The IMAGE_DEF block must be first in flash.  The vector table must be within the first 4 KB, aligned to 128 bytes.  All code goes into flash.  The stack lives in SRAM.  If any of these placements are wrong, the boot ROM will not recognize the binary, or the CPU will jump to the wrong address.
