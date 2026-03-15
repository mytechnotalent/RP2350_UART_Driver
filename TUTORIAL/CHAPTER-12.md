# Chapter 12: ARM Jumps, Calls, and Returns

## Introduction

Functions are the building blocks of our firmware.  This chapter explains how ARM Cortex-M33 implements function calls and returns, how the link register works, how the stack preserves the call chain, and how our firmware's call graph is structured.

## The bl Instruction — Function Call

```asm
  bl    UART_Init                                // call UART_Init
```

`bl` (branch with link) performs two actions in one instruction:
1. **Save return address**: stores the address of the next instruction (`bl` + 4) into LR (r14)
2. **Branch**: sets PC to the address of `UART_Init`

The processor continues executing at the first instruction of `UART_Init`.

## The bx lr Instruction — Function Return

```asm
  bx    lr                                       // return to caller
```

`bx lr` sets PC to the value in LR, resuming execution at the instruction after the original `bl`.  The lowest bit of LR is 1 (indicating Thumb mode), which `bx` uses to stay in Thumb state.

## The Complete Call/Return Sequence

```asm
  // In Reset_Handler:
  bl    Init_XOSC                                // LR = address of next instruction
  bl    Enable_XOSC_Peri_Clock                   // LR = address of next instruction
  ...

  // In Init_XOSC:
  ldr   r0, =XOSC_STARTUP                        // first instruction of Init_XOSC
  ...
  bx    lr                                       // return to Reset_Handler
```

## The Problem: Nested Calls

When a function calls another function, the second `bl` overwrites LR.  The original return address is lost:

```asm
  // Reset_Handler calls Init_Stack
  bl    Init_Stack                               // LR = return address in Reset_Handler

  // If Init_Stack called another function:
  bl    some_helper                              // LR would be overwritten!
  bx    lr                                       // would NOT return to Reset_Handler
```

## The Solution: push/pop

Functions that call other functions save LR on the stack:

```asm
UART0_Out:
  push  {r4-r12, lr}                             // save lr and callee-saved registers
  // ... function body ...
  pop   {r4-r12, lr}                             // restore lr
  bx    lr                                       // return using restored lr
```

`push {r4-r12, lr}` saves the return address on the stack.  `pop {r4-r12, lr}` restores it.  This allows arbitrarily deep call chains.

## Leaf Functions vs Non-Leaf Functions

| Type | Definition | push/pop needed? |
|---|---|---|
| Leaf | Does not call other functions | No |
| Non-leaf | Calls other functions with `bl` | Yes |

Our leaf functions (no nested calls):
- `Init_Stack` — sets stack pointers and returns
- `Init_XOSC` — configures oscillator and returns
- `Enable_XOSC_Peri_Clock` — sets clock and returns
- `Init_Subsystem` — releases reset and returns
- `UART_Release_Reset` — releases UART reset and returns
- `UART_Init` — configures UART and returns
- `Enable_Coprocessor` — enables coprocessor and returns

Our non-leaf functions (must save LR):
- `main` — calls `UART0_In` and `UART0_Out`
- `UART0_Out` — saves/restores context (though currently a leaf, it follows the convention)
- `UART0_In` — saves/restores context

## The b Instruction — Tail Call

```asm
  b     main                                     // branch to main (no return expected)
```

In `Reset_Handler`, the final instruction is `b main`, not `bl main`.  This is a tail call — we jump to main without saving a return address, because main never returns (it loops forever).

## The Call Graph

```
  Power On
     |
     v
  Vector Table → Reset_Handler
                     |
                     +→ bl Init_Stack
                     +→ bl Init_XOSC
                     +→ bl Enable_XOSC_Peri_Clock
                     +→ bl Init_Subsystem
                     +→ bl UART_Release_Reset
                     +→ bl UART_Init
                     +→ bl Enable_Coprocessor
                     +→ b  main
                            |
                            +→ bl UART0_In  (loops forever)
                            +→ bl UART0_Out
```

Reset_Handler calls seven initialization functions, then jumps to main.  Main calls UART0_In and UART0_Out in an infinite loop.

## The Thumb Bit

ARM Cortex-M33 always runs in Thumb state.  When calling a function, the target address must have bit 0 set to 1.  The `bl` instruction handles this automatically.  In the vector table, we explicitly set the Thumb bit:

```asm
  .word Reset_Handler + 1                        // Thumb bit set
```

## Summary

`bl` calls a function by saving the return address in LR and branching to the target.  `bx lr` returns by branching to the address in LR.  Functions that make calls must save LR on the stack with `push` and restore it with `pop`.  `b` is used for unconditional jumps when no return is needed (tail calls, infinite loops).  Our firmware has a simple call graph: Reset_Handler calls seven init functions, then main loops forever calling UART0_In and UART0_Out.
