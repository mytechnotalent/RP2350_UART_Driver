# Chapter 17: Memory-Mapped I/O — Controlling Hardware Through Addresses

## Introduction

On the RP2350, there is no special I/O bus or port instruction.  Every peripheral — UART, GPIO, clocks, resets — is controlled by reading and writing specific memory addresses.  The processor does not know the difference between accessing RAM and accessing a UART register.  It uses the same `ldr` and `str` instructions for both.  This is memory-mapped I/O.

## How Memory-Mapped I/O Works

Each peripheral occupies a range of addresses in the memory map.  Within that range, each register has a fixed offset.  To configure a peripheral, you write specific values to specific addresses.

```asm
  ldr   r0, =0x40070000                          // UART0 base address
  ldr   r1, =6                                   // baud rate integer divisor
  str   r1, [r0, #0x24]                          // write to UART0 + 0x24 (UARTIBRD)
```

The processor executes `str` exactly as it would for a RAM write, but the bus fabric routes the write to the UART peripheral instead of SRAM.

## Reading vs Writing Peripheral Registers

| Operation | Instruction | Effect |
|---|---|---|
| Read status | `ldr r1, [r0]` | Returns current hardware state |
| Write config | `str r1, [r0]` | Changes hardware behavior |
| Read-modify-write | ldr, bic/orr, str | Changes specific bits |

**Reading** a status register returns real-time hardware state.  Each `ldr` returns a fresh value — the hardware may change between reads.

**Writing** a configuration register causes the hardware to respond immediately.  Writing UARTCR with the enable bit set turns the UART on.

## The RP2350 Peripheral Address Map

| Peripheral | Base Address | Description |
|---|---|---|
| XOSC | 0x40048000 | Crystal oscillator |
| CLOCKS | 0x40010000 | Clock controller |
| RESETS | 0x40020000 | Reset controller |
| IO_BANK0 | 0x40028000 | GPIO function select |
| PADS_BANK0 | 0x40038000 | GPIO pad control |
| UART0 | 0x40070000 | First UART peripheral |
| PPB | 0xE0000000 | Private peripheral bus |

Each base address is the starting address of a block of registers.  Individual registers are accessed at `base + offset`.

## UART0 Register Map

| Register | Offset | Address | Purpose |
|---|---|---|---|
| UARTDR | 0x00 | 0x40070000 | Data (read = RX, write = TX) |
| UARTFR | 0x18 | 0x40070018 | Flag register (FIFO status) |
| UARTIBRD | 0x24 | 0x40070024 | Integer baud rate divisor |
| UARTFBRD | 0x28 | 0x40070028 | Fractional baud rate divisor |
| UARTLCR_H | 0x2C | 0x4007002C | Line control (word length, FIFO) |
| UARTCR | 0x30 | 0x40070030 | Control (enable, TX/RX enable) |

## Volatile Behavior

Hardware registers are volatile — their values can change without the CPU writing to them.  For example, UARTFR bit 5 (TX FIFO full) changes when the FIFO hardware transmits a byte.

This is why polling loops must re-read the register on every iteration:

```asm
.poll:
  ldr   r5, [r4, #0x18]                          // read UARTFR (fresh value each time)
  ldr   r6, =32                                  // bit 5 mask
  ands  r5, r5, r6                               // test TX FIFO full
  bne   .poll                                    // loop if full
```

Each `ldr` fetches the current state of the hardware.  The loop exits only when the hardware clears the TXFF bit (FIFO has space).

## Atomic Aliases

The RP2350 provides atomic access aliases for some registers.  For example, RESETS_RESET at 0x40020000 has a clear alias at 0x40023000.  Writing to the clear alias atomically clears the specified bits without reading first:

```
  Write to 0x40023000 with (1<<6) → clears bit 6 of RESETS_RESET
```

Our firmware uses explicit read-modify-write instead, but the atomic aliases exist as an alternative.

## Why Not Use Special I/O Instructions?

Some architectures (x86) have dedicated `in` and `out` instructions for I/O.  ARM does not.  Memory-mapped I/O has advantages:

1. **Simplicity**: no separate I/O instruction set to learn
2. **Uniformity**: same load/store instructions for everything
3. **Flexibility**: peripherals can use the full addressing capability of the processor
4. **C compatibility**: peripheral registers look like regular pointers in C

## Barriers: dsb and isb

When writing to system-level registers (like CPACR in the PPB), the processor may need barriers to ensure the write takes effect before the next instruction:

```asm
  str   r1, [r0]                                 // write to CPACR
  dsb                                            // ensure write completes
  isb                                            // flush instruction pipeline
```

`dsb` (Data Synchronization Barrier) ensures all memory accesses before it complete before any after it begin.  `isb` (Instruction Synchronization Barrier) flushes the pipeline so the processor fetches fresh instructions with the new configuration active.

Our firmware uses these after enabling the coprocessor.

## Summary

Memory-mapped I/O is the universal mechanism for hardware control on ARM.  Every peripheral register has a unique memory address.  Reading an address returns the hardware's current state.  Writing an address changes the hardware's configuration.  The same `ldr`/`str` instructions used for RAM access work for peripheral registers.  Hardware registers are volatile — they can change between reads.  Barriers (`dsb`, `isb`) ensure writes take effect when configuring system-level resources.
