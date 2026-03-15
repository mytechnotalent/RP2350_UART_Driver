# Chapter 14: Assembler Directives — Controlling the Assembly Process

## Introduction

Assembler directives are commands to the assembler — they do not generate machine instructions.  Instead, they control how the assembler organizes output, defines constants, aligns data, and manages sections.  This chapter covers every directive used in our firmware.

## .syntax unified

```asm
  .syntax unified                                // use unified assembly syntax
```

This directive tells the GNU assembler to use the Unified Assembly Language (UAL).  UAL allows the same syntax for both ARM and Thumb instructions.  Without this, you would need different syntax for each mode.  Every source file in our project starts with this directive.

## .cpu cortex-m33

```asm
  .cpu cortex-m33                                // target Cortex-M33 core
```

Tells the assembler which CPU to target.  This determines which instructions are valid.  Cortex-M33 supports the ARMv8-M Mainline instruction set, including hardware divide, DSP extensions, and optional TrustZone.

## .thumb

```asm
  .thumb                                         // use Thumb instruction set
```

Forces the assembler to generate Thumb-2 instructions.  ARM Cortex-M33 can only execute Thumb instructions — it does not support ARM (32-bit) mode.  This directive is essential.

## .section — Define Output Section

```asm
  .section .text                                 // code section
  .section .vectors, "ax"                        // vector table section
  .section .picobin_block, "a"                   // boot metadata section
  .section .rodata                               // read-only data section
  .section .data                                 // initialized data section
  .section .bss                                  // uninitialized data section
```

Sections organize the output into logical groups.  The linker script (Chapter 19) decides where each section is placed in the final binary:

| Section | Flags | Contents | Placement |
|---|---|---|---|
| .picobin_block | "a" (alloc) | IMAGE_DEF boot metadata | First in flash |
| .vectors | "ax" (alloc, exec) | Vector table | 128-byte aligned in flash |
| .text | — | Executable code | After vectors in flash |
| .rodata | — | Read-only data | After .text in flash |
| .data | — | Initialized globals | Not used in our firmware |
| .bss | — | Uninitialized globals | Not used in our firmware |

## .align — Alignment

```asm
  .align 2                                       // align to 4-byte boundary
```

`.align 2` means align to 2² = 4 bytes.  The assembler inserts padding bytes (NOP or zero) until the current address is divisible by 4.

Word-aligned code is required on ARM Cortex-M33.  Every function should start on a 4-byte boundary.

## .equ — Define a Constant

```asm
  .equ UART0_BASE, 0x40070000
```

Creates a named constant.  `UART0_BASE` is replaced by `0x40070000` everywhere it appears.  This is a pure assembly-time substitution — no memory is allocated.

Our `constants.s` file is entirely `.equ` directives, defining every address and constant used across all source files.

### Computed Constants

Constants can reference other constants:

```asm
  .equ XOSC_BASE,    0x40048000
  .equ XOSC_CTRL,    XOSC_BASE + 0x00            // = 0x40048000
  .equ XOSC_STATUS,  XOSC_BASE + 0x04            // = 0x40048004
  .equ XOSC_STARTUP, XOSC_BASE + 0x0c            // = 0x4004800C
```

The assembler evaluates these at assembly time.

## .global — Export Symbol

```asm
  .global UART_Init                              // linker can see this symbol
```

Without `.global`, a label is only visible within its source file.  Each function we define is exported with `.global` so other files can call it.

## .type — Symbol Type

```asm
  .type UART_Init, %function                     // mark as function
```

This metadata:
1. Sets the Thumb bit (bit 0) on the symbol's address
2. Enables proper `bl` targeting
3. Helps debuggers show function boundaries

## .size — Symbol Size

```asm
  .size Reset_Handler, . - Reset_Handler         // function size in bytes
```

Records the size of the function for debugging and analysis.  Only used on `Reset_Handler` in our firmware.

## .word — Emit 32-bit Value

```asm
  .word STACK_TOP                                // emit 0x20082000
  .word Reset_Handler + 1                        // emit address with Thumb bit
```

Places a raw 32-bit value at the current position in the output.  Used to build the vector table.

## .byte / .hword — Emit Smaller Values

```asm
  .byte  0x42                                    // emit 1 byte
  .hword 0x0001                                  // emit 2 bytes (halfword)
```

Used in `image_def.s` to construct the PICOBIN block with precise byte-level control.

## .include — File Inclusion

```asm
  .include "constants.s"                         // insert constants.s here
```

The assembler replaces this line with the entire contents of `constants.s`.  After this, all `.equ` definitions from `constants.s` are available.

## KEEP in Linker Context

The `.section` flags interact with the linker.  `KEEP(*)` in the linker script prevents the linker from discarding sections that have no references:

```
  KEEP(*(.vectors))
  KEEP(*(.picobin_block))
```

Without `KEEP`, the linker might remove the vector table because no code explicitly references it — the hardware does.

## Summary of All Directives Used

| Directive | Purpose |
|---|---|
| `.syntax unified` | Use UAL syntax |
| `.cpu cortex-m33` | Target Cortex-M33 |
| `.thumb` | Generate Thumb-2 code |
| `.section` | Set current output section |
| `.align` | Insert padding for alignment |
| `.equ` | Define named constant |
| `.global` | Export symbol to linker |
| `.type` | Set symbol type (function/object) |
| `.size` | Record symbol size |
| `.word` | Emit 32-bit value |
| `.byte` | Emit 8-bit value |
| `.hword` | Emit 16-bit value |
| `.include` | Insert another file |

## Summary

Directives control how the assembler organizes and emits your code.  `.syntax unified`, `.cpu`, and `.thumb` configure the assembler.  `.section` and `.align` organize output.  `.equ` creates named constants.  `.global` and `.type` make symbols accessible.  `.word`, `.byte`, and `.hword` emit raw data.  Every directive in our firmware serves a specific, necessary purpose.
