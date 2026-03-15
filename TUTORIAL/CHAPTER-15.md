# Chapter 15: The Calling Convention and Stack Frames

## Introduction

When one function calls another, both functions share the same 16 registers.  Without rules, the called function could destroy values the caller still needs.  The **calling convention** is a set of rules that prevents this chaos.  Every function in our firmware follows these rules.

## The ARM AAPCS Calling Convention

The ARM Architecture Procedure Call Standard (AAPCS) defines how functions pass arguments, return values, and preserve registers.

### Argument Passing

Arguments are passed in registers r0 through r3:
- First argument → r0
- Second argument → r1
- Third argument → r2
- Fourth argument → r3
- Additional arguments go on the stack (rare in embedded)

### Return Values

Return values use r0 and r1:
- Single 32-bit return → r0
- 64-bit return → r0 (low), r1 (high)

### Register Preservation Rules

| Category | Registers | Rule |
|---|---|---|
| Caller-saved | r0-r3, r12 | Callee may overwrite freely |
| Callee-saved | r4-r11 | Callee must save and restore |
| Special | r13/SP | Must be preserved, 8-byte aligned |
| Special | r14/LR | Saved if function makes calls |
| Special | r15/PC | Program counter |

## Caller-Saved Registers: r0-r3, r12

After calling a function, the caller must assume r0-r3 and r12 are destroyed.  If the caller needs a value in r0 after the call, it must save it before and restore it after.

In our firmware, `UART0_In` returns the received byte in r0.  The next call to `UART0_Out` uses r0 as the argument — the return value from In becomes the argument to Out:

```asm
  bl    UART0_In                                 // r0 = received byte
  bl    UART0_Out                                // transmit byte in r0
```

## Callee-Saved Registers: r4-r11

If a function uses registers r4 through r11, it must restore their original values before returning.  This is done with push/pop:

```asm
UART0_Out:
  push  {r4-r12, lr}                             // save callee-saved registers + lr
  // ... use r4, r5, r6 freely ...
  pop   {r4-r12, lr}                             // restore original values
  bx    lr                                       // return
```

### Why Save r4-r12?

Consider this scenario without saving:

```
  main:
    mov   r4, #42         ; r4 = 42
    bl    UART0_Out
    ; r4 should still be 42 here!
```

If `UART0_Out` used r4 without saving it, r4 would no longer be 42 when control returns to main.  This would be a bug.  The push/pop at entry/exit guarantees r4-r11 are unchanged from the caller's perspective.

## The Stack Frame

When a function executes `push {r4-r12, lr}`, it creates a **stack frame**:

```
  Before push:
    SP → [empty]

  After push {r4-r12, lr}:
    SP → [r4 ]  SP + 0
         [r5 ]  SP + 4
         [r6 ]  SP + 8
         [r7 ]  SP + 12
         [r8 ]  SP + 16
         [r9 ]  SP + 20
         [r10]  SP + 24
         [r11]  SP + 28
         [r12]  SP + 32
         [lr ]  SP + 36
```

The stack grows downward.  SP decreases by 40 bytes (10 registers × 4 bytes).

## Function Prologue and Epilogue

| Part | Instructions | Purpose |
|---|---|---|
| Prologue | `push {r4-r12, lr}` | Save registers |
| Body | ... computation ... | Function work |
| Epilogue | `pop {r4-r12, lr}` | Restore registers |
| Return | `bx lr` | Return to caller |

## Leaf vs Non-Leaf Functions

### Leaf Function (No push/pop needed if only using r0-r3)

```asm
Init_XOSC:
  ldr   r0, =XOSC_STARTUP                        // uses r0, r1 only
  ldr   r1, =0x00c4                              // no callee-saved registers used
  str   r1, [r0]                                 // no nested calls
  ...
  bx    lr                                       // direct return
```

`Init_XOSC` only uses r0 and r1 (caller-saved), makes no nested calls, so it does not need push/pop.

### Non-Leaf Function (Must save LR)

```asm
main:
  push  {r4-r12, lr}                             // save lr (will be overwritten by bl)
  bl    UART0_In                                 // this overwrites lr
  bl    UART0_Out                                // this overwrites lr again
  ...
  pop   {r4-r12, lr}                             // restore lr
  bx    lr                                       // return
```

`main` calls other functions with `bl`, which overwrites LR.  It must save LR on entry and restore it on exit.

## Our Firmware's Functions

| Function | Leaf? | push/pop? | Why |
|---|---|---|---|
| Init_Stack | Yes | No | Uses r0 only |
| Init_XOSC | Yes | No | Uses r0, r1 only |
| Enable_XOSC_Peri_Clock | Yes | No | Uses r0, r1 only |
| Init_Subsystem | Yes | No | Uses r0, r1 only |
| UART_Release_Reset | Yes | No | Uses r0, r1 only |
| UART_Init | Yes | No | Uses r0, r1 only |
| Enable_Coprocessor | Yes | No | Uses r0, r1 only |
| Reset_Handler | No | No | Tail-calls main via `b` |
| UART0_Out | Yes | Yes | Convention: uses r4-r6 |
| UART0_In | Yes | Yes | Convention: uses r4-r6 |
| main | No | Yes | Calls UART0_In/Out |
| Delay_MS | Yes | Yes | Uses r4-r5 |

## Summary

The AAPCS calling convention defines which registers are preserved across calls.  r0-r3 are caller-saved (used for arguments and return values).  r4-r11 are callee-saved (must be preserved by push/pop).  LR is saved when a function makes nested calls.  Every function in our firmware follows these rules, ensuring registers are not silently corrupted across function boundaries.
