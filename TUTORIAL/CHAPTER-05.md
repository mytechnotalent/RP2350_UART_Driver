# Chapter 5: Load-Store Architecture — How ARM Accesses Memory

## Introduction

ARM Cortex-M33 is a load-store architecture.  This means the processor cannot operate on data in memory directly.  To add two numbers stored in memory, you must: (1) load the first number into a register, (2) load the second number into a register, (3) add the two registers, (4) store the result back to memory.  Only load and store instructions touch memory.  Every other instruction operates exclusively on registers.

## Why Load-Store?

Load-store architecture simplifies the processor design.  The CPU has two separate paths:

1. **Memory path**: `ldr` (load) and `str` (store) move data between registers and memory
2. **Computation path**: `add`, `sub`, `orr`, `and`, etc. operate on registers only

This separation makes the pipeline efficient: while one instruction is computing, another can be loading or storing.

## The Load Instruction: ldr

`ldr` copies a word (4 bytes) from memory into a register:

```asm
  ldr   r1, [r0]                                 // r1 = word at address in r0
```

The square brackets `[r0]` mean "the memory address contained in r0."  If r0 = 0x40070000, then `ldr r1, [r0]` reads the 32-bit value at memory address 0x40070000 into r1.

### Load with Offset

You can add a constant offset to the base address:

```asm
  ldr   r1, [r0, #0x18]                          // r1 = word at (r0 + 0x18)
```

If r0 = 0x40070000, this reads from 0x40070018 — the UART Flag Register.

### Load Pseudo-Instruction

The `ldr r0, =value` form is a pseudo-instruction.  The assembler places the constant in a literal pool and generates a PC-relative load:

```asm
  ldr   r0, =0x40070000                          // r0 = 0x40070000
```

This is necessary because ARM Thumb-2 instructions cannot encode arbitrary 32-bit constants directly.  The assembler handles the details.

## The Store Instruction: str

`str` copies a word from a register into memory:

```asm
  str   r1, [r0]                                 // word at address in r0 = r1
```

### Store with Offset

```asm
  str   r1, [r0, #0x30]                          // word at (r0 + 0x30) = r1
```

If r0 = 0x40070000, this writes r1 to address 0x40070030 — the UART Control Register.

## The Load-Modify-Store Pattern

Almost every hardware configuration in our firmware follows this pattern:

```asm
  ldr   r0, =RESETS_RESET                        // load register address
  ldr   r1, [r0]                                 // READ: load current value
  bic   r1, r1, #(1<<26)                         // MODIFY: clear bit 26
  str   r1, [r0]                                 // WRITE: store modified value
```

This is called Read-Modify-Write (RMW).  You read the current register value, change the specific bits you need, and write it back.  This preserves all other bits in the register.

## Byte and Halfword Access

ARM supports smaller loads and stores:

```asm
  ldrb  r1, [r0]                                 // load byte (8 bits, zero-extended)
  ldrh  r1, [r0]                                 // load halfword (16 bits, zero-extended)
  strb  r1, [r0]                                 // store byte (lowest 8 bits)
  strh  r1, [r0]                                 // store halfword (lowest 16 bits)
```

Our firmware only uses word-sized accesses because all RP2350 peripheral registers are 32 bits wide.

## Push and Pop

Push and pop are special forms of store and load that use the stack pointer:

```asm
  push  {r4-r12, lr}                             // store r4-r12 and lr to stack
  pop   {r4-r12, lr}                             // load r4-r12 and lr from stack
```

`push` decrements SP by 4 for each register, then stores each register.  `pop` loads each register, then increments SP.  This implements the function prologue/epilogue for saving and restoring callee-saved registers.

## Memory Access in Our Firmware

Every peripheral interaction in our code is a load, a store, or both:

| Operation | Instruction | Example |
|---|---|---|
| Read register value | `ldr r1, [r0]` | Read UARTFR |
| Write register value | `str r1, [r0]` | Write UARTDR |
| Read at offset | `ldr r1, [r0, #0x18]` | Read UART0+0x18 |
| Write at offset | `str r1, [r0, #0x30]` | Write UART0+0x30 |
| Load address | `ldr r0, =ADDR` | Load constant into register |
| Save context | `push {r4-r12, lr}` | Save registers to stack |
| Restore context | `pop {r4-r12, lr}` | Restore registers from stack |

## Summary

ARM Cortex-M33 is a load-store architecture.  Only `ldr` and `str` (and their variants) access memory.  All computation happens in registers.  The load-modify-store pattern (read-modify-write) is the fundamental technique for configuring hardware peripherals.  Push and pop are specialized stack operations that save and restore register state across function calls.
