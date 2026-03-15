# Chapter 6: The Fetch-Decode-Execute Cycle in Detail

## Introduction

Chapter 1 introduced the fetch-decode-execute cycle as the heartbeat of the CPU.  This chapter examines exactly what happens at each stage inside the ARM Cortex-M33 core, how the pipeline works, and how the cycle relates to the instructions you will write.

## The Three Stages

### Stage 1: Fetch

The processor reads the instruction at the address held in the Program Counter (PC / r15).  On the RP2350, this means reading from flash (XIP) at addresses starting at 0x10000000:

```
  PC = 0x100000A0
  Processor reads bytes at 0x100000A0 from flash cache
  Instruction bytes are captured into the instruction register
  PC advances to 0x100000A2 or 0x100000A4 (depending on 16-bit or 32-bit instruction)
```

ARM Thumb-2 instructions are either 16 bits (2 bytes) or 32 bits (4 bytes).  The processor examines the first halfword to determine the instruction length.

### Stage 2: Decode

The control unit examines the instruction bits and determines:
- What operation to perform (add, load, branch, etc.)
- Which registers are the source operands
- Which register is the destination
- Whether there is an immediate value embedded in the instruction
- Whether the condition flags need to be updated

### Stage 3: Execute

The processor performs the operation:
- For arithmetic: the ALU computes the result
- For loads: the memory system fetches data from the address
- For stores: the memory system writes data to the address
- For branches: the PC is updated to the target address

## The Pipeline

ARM Cortex-M33 uses a pipeline so that multiple instructions are in different stages simultaneously:

```
  Clock cycle:    1       2       3       4       5
  Instruction 1:  FETCH   DECODE  EXECUTE
  Instruction 2:          FETCH   DECODE  EXECUTE
  Instruction 3:                  FETCH   DECODE  EXECUTE
```

In ideal conditions, one instruction completes every clock cycle, even though each instruction takes three cycles to complete.  This is possible because different instructions are at different stages at the same time.

### Pipeline Hazards

Sometimes the pipeline stalls:
- **Data hazard**: an instruction needs a value that the previous instruction has not finished computing
- **Branch hazard**: a branch changes PC, so the already-fetched instructions are wrong and must be discarded (pipeline flush)

This is why branches are relatively expensive on pipelined processors.  Our polling loops (waiting for hardware registers) involve many branches, but since we are waiting for slow hardware anyway, pipeline efficiency is irrelevant in those loops.

## A Concrete Example

Consider this code from our UART receive function:

```asm
  ldr   r4, =UART0_BASE                          // FETCH-DECODE-EXECUTE: r4 = 0x40070000
  ldr   r5, [r4, #0x18]                          // FETCH-DECODE-EXECUTE: r5 = UARTFR value
  ldr   r6, =16                                  // FETCH-DECODE-EXECUTE: r6 = 16 (bit 4 mask)
  ands  r5, r5, r6                               // FETCH-DECODE-EXECUTE: r5 = r5 AND 16
  bne   .UART0_In_loop                           // FETCH-DECODE-EXECUTE: branch if not zero
```

Each line goes through all three stages.  The `ldr r5, [r4, #0x18]` instruction must wait until `ldr r4, =UART0_BASE` has completed execution so that r4 contains the correct address.  The `ands` must wait for `ldr r5` and `ldr r6` to complete.  The `bne` must wait for `ands` to set the condition flags.

## How Branch Instructions Affect the Pipeline

When `bne .UART0_In_loop` decides to branch (because the Z flag is not set), the processor:
1. Discards any instructions that were fetched after the `bne`
2. Sets PC to the address of `.UART0_In_loop`
3. Starts fetching from the new address

This "pipeline flush" costs a few clock cycles.  On Cortex-M33, the penalty is small (1-2 cycles) because the pipeline is short.

## The Cortex-M33 Execution Model

ARM Cortex-M33 operates in two modes:
- **Thread mode**: normal program execution (our main code)
- **Handler mode**: exception/interrupt handling

Our firmware runs entirely in Thread mode with MSP as the stack pointer.  We do not use interrupts — we use polling (busy-waiting on hardware status registers).

## Clock Speed

On the RP2350, the Cortex-M33 core runs at up to 150 MHz after clock configuration.  Before we configure the XOSC and peripheral clock, the CPU runs from a slower internal oscillator.  Each clock tick advances the pipeline by one stage.

At 150 MHz, one clock cycle is 6.67 nanoseconds.  Most instructions execute in one cycle.  Loads from peripheral registers may take multiple cycles due to the APB bus bridge.

## Summary

The fetch-decode-execute cycle is the fundamental operation of the ARM Cortex-M33 core.  The pipeline allows multiple instructions to be in flight simultaneously, completing approximately one per clock tick in ideal conditions.  Branches flush the pipeline and cost a few extra cycles.  ARM Thumb-2 instructions can be 16 or 32 bits.  The processor runs in Thread mode for normal code and Handler mode for exceptions.  Understanding the pipeline explains why our code is structured the way it is — each instruction depends on the results of previous instructions, and the hardware enforces these dependencies automatically.
