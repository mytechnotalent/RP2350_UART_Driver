# Chapter 4: What Is a Register?

## Introduction

A register is a tiny, fast storage location inside the processor.  Registers are the only place where the CPU can perform operations.  You cannot add two values in memory directly — you must first load them into registers, perform the operation, and store the result back.  This chapter explains every register in the ARM Cortex-M33 core.

## The ARM Cortex-M33 Register File

ARM Cortex-M33 has 16 general-purpose registers plus several special registers:

```
  +--------+--------+-------------------------------------------+
  | Name   | Number | Purpose                                   |
  +--------+--------+-------------------------------------------+
  | r0     | x0     | Argument / return value / scratch         |
  | r1     | x1     | Argument / return value / scratch         |
  | r2     | x2     | Argument / scratch                        |
  | r3     | x3     | Argument / scratch                        |
  | r4     | x4     | Callee-saved (preserved across calls)     |
  | r5     | x5     | Callee-saved                              |
  | r6     | x6     | Callee-saved                              |
  | r7     | x7     | Callee-saved                              |
  | r8     | x8     | Callee-saved                              |
  | r9     | x9     | Callee-saved                              |
  | r10    | x10    | Callee-saved                              |
  | r11    | x11    | Callee-saved (frame pointer)              |
  | r12    | x12    | Intra-procedure scratch (IP)              |
  | r13/SP | x13    | Stack Pointer                             |
  | r14/LR | x14    | Link Register (return address)            |
  | r15/PC | x15    | Program Counter (current instruction)     |
  +--------+--------+-------------------------------------------+
```

Each register is 32 bits wide.  That means each register can hold one word — a value from 0x00000000 to 0xFFFFFFFF.

## Registers r0-r3: Arguments and Scratch

These are caller-saved registers used for passing arguments to functions and returning values:

- **r0**: first argument, also the return value
- **r1**: second argument (or upper half of 64-bit return)
- **r2**: third argument
- **r3**: fourth argument

After a function call, these registers may contain anything.  The caller must assume they are destroyed.

## Registers r4-r11: Callee-Saved

These registers are preserved across function calls.  If a function uses any of r4-r11, it must save the original values on the stack at entry and restore them before returning.

Our firmware does this with `push {r4-r12, lr}` at function entry and `pop {r4-r12, lr}` at function exit.

## Register r12 (IP): Intra-Procedure Scratch

r12 is used by the linker for veneers (branch stubs) and can be freely used as scratch within a function.  Our firmware saves it alongside r4-r11 as a convention.

## Register r13 (SP): Stack Pointer

The stack pointer holds the address of the top of the stack.  ARM Cortex-M33 has two stack pointers:

- **MSP** (Main Stack Pointer): used in handler mode (interrupts) and by default
- **PSP** (Process Stack Pointer): used in thread mode (optional)

Our firmware initializes both to STACK_TOP (0x20082000).  The stack grows downward — pushing decreases SP, popping increases SP.

## Register r14 (LR): Link Register

When you call a function with `bl`, the processor stores the return address in LR (r14).  The called function returns by branching to LR with `bx lr`.

If the called function itself calls another function, it must save LR on the stack first (via `push {lr}`) because the nested `bl` will overwrite LR.

## Register r15 (PC): Program Counter

The program counter holds the address of the current instruction (technically, the current instruction + 4 in ARM due to the pipeline).  You rarely write to PC directly.  Branches and function calls modify PC implicitly.

## Special Registers

ARM Cortex-M33 has special registers accessed with `msr` and `mrs` instructions:

| Register | Purpose |
|---|---|
| MSP | Main Stack Pointer |
| PSP | Process Stack Pointer |
| MSPLIM | MSP lower limit (stack overflow detection) |
| PSPLIM | PSP lower limit |
| PRIMASK | Interrupt disable bit |
| CONTROL | Privilege and stack selection |
| xPSR | Combined program status register |

Our firmware sets MSP, PSP, MSPLIM, and PSPLIM in the Init_Stack function:

```asm
  ldr   r0, =STACK_TOP                           // load stack top value
  msr   PSP, r0                                  // set process stack pointer
  ldr   r0, =STACK_LIMIT                         // load stack limit value
  msr   MSPLIM, r0                               // set MSP lower limit
  msr   PSPLIM, r0                               // set PSP lower limit
  ldr   r0, =STACK_TOP                           // reload stack top value
  msr   MSP, r0                                  // set main stack pointer
```

## The Program Status Register (xPSR)

The xPSR is actually three registers combined:

- **APSR**: Application PSR — contains condition flags (N, Z, C, V)
- **IPSR**: Interrupt PSR — contains the active exception number
- **EPSR**: Execution PSR — contains the Thumb bit (always 1 on Cortex-M)

The condition flags in APSR are critical for branches:

| Flag | Name | Meaning |
|---|---|---|
| N | Negative | Result was negative (bit 31 set) |
| Z | Zero | Result was zero |
| C | Carry | Unsigned overflow or shift carry-out |
| V | Overflow | Signed overflow |

Instructions like `tst`, `cmp`, and `subs` set these flags.  Conditional branches like `beq` and `bne` read them.

## Register Usage in Our Firmware

Looking at our actual code:

```asm
  ldr   r0, =RESETS_RESET                        // r0 = address (argument to load)
  ldr   r1, [r0]                                 // r1 = value at that address
  bic   r1, r1, #(1<<26)                         // r1 = value with bit 26 cleared
  str   r1, [r0]                                 // write modified value back
```

Here r0 holds an address, r1 holds data.  We load, modify, and store — the fundamental pattern of all hardware programming on ARM.

## Summary

ARM Cortex-M33 has 16 general-purpose registers (r0-r15), each 32 bits wide.  r0-r3 pass arguments and return values.  r4-r11 are preserved across calls.  r13 is the stack pointer, r14 is the link register (return address), and r15 is the program counter.  Special registers like MSP, PSP, and their limits are accessed with `msr`/`mrs`.  The condition flags (N, Z, C, V) in xPSR control conditional branching.  All computation happens in registers — memory is accessed only through load and store instructions.
