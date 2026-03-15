# Chapter 21: image_def.s — The PICOBIN Block Byte by Byte

## Introduction

Every RP2350 binary must contain a valid IMAGE_DEF block in the first 4 KB of flash.  The boot ROM scans for this metadata to determine whether flash contains a valid program.  Without it, the boot ROM treats flash as blank or corrupt and will not boot.

This chapter explains every byte of the IMAGE_DEF block in `image_def.s`.

## Full Source: image_def.s

```
.section .picobin_block, "a"                     // place IMAGE_DEF block in flash

.word  0xffffded3                                // PICOBIN_BLOCK_MARKER_START
.byte  0x42                                      // PICOBIN_BLOCK_ITEM_1BS_IMAGE_TYPE
.byte  0x1                                       // item is 1 word in size
.hword 0b0001000000100001                        // SECURE mode (0x1021)
.byte  0xff                                      // PICOBIN_BLOCK_ITEM_2BS_LAST
.hword 0x0001                                    // item is 1 word in size
.byte  0x0                                       // pad
.word  0x0                                       // relative pointer to next block (0 = loop to self)
.word  0xab123579                                // PICOBIN_BLOCK_MARKER_END
```

## Line-by-Line Walkthrough

### Section Directive

```
.section .picobin_block, "a"                     // place IMAGE_DEF block in flash
```

Creates a section named `.picobin_block` with the `"a"` flag (allocatable — occupies space in the output binary).  The linker script places this section first in flash via the `.embedded_block` output section.

### Marker Start

```
.word  0xffffded3                                // PICOBIN_BLOCK_MARKER_START
```

`.word` emits a 32-bit value.  `0xffffded3` is the magic number the boot ROM looks for when scanning flash.  If the boot ROM does not find this pattern in the first 4 KB, it will not attempt to boot from flash.

In memory (little-endian):

```
  Address      Byte
  ----------   ----
  0x10000000   0xd3
  0x10000001   0xde
  0x10000002   0xff
  0x10000003   0xff
```

### Image Type Item — Tag Byte

```
.byte  0x42                                      // PICOBIN_BLOCK_ITEM_1BS_IMAGE_TYPE
```

`.byte` emits a single 8-bit value.  `0x42` encodes two fields:
- The low nibble `0x2` = IMAGE_TYPE item kind
- The high nibble `0x4` = 1BS (one-byte-size) encoding

This tells the boot ROM that the next byte is a size field and the data describes the image type.

### Image Type Item — Size

```
.byte  0x1                                       // item is 1 word in size
```

The item payload is 1 word (4 bytes) long.

### Image Type Item — Payload

```
.hword 0b0001000000100001                        // SECURE mode (0x1021)
```

`.hword` emits a 16-bit value.  The binary value `0b0001000000100001` = `0x1021`:
- Bits 3:0 = `0x1` — EXE (executable image)
- Bits 7:4 = `0x2` — security: Secure mode
- Bits 15:8 = `0x10` — ARM CPU (as opposed to RISC-V)

This tells the boot ROM: "This is an ARM Secure executable."

### Last Item — Tag Byte

```
.byte  0xff                                      // PICOBIN_BLOCK_ITEM_2BS_LAST
```

`0xff` marks this as the last item in the block.  The boot ROM stops parsing items after seeing this tag.

### Last Item — Size

```
.hword 0x0001                                    // item is 1 word in size
```

The last item has a 1-word payload.

### Last Item — Pad

```
.byte  0x0                                       // pad
```

Padding byte to maintain 4-byte alignment within the block structure.

### Next Block Pointer

```
.word  0x0                                       // relative pointer to next block (0 = loop to self)
```

A relative offset to the next PICOBIN block.  Zero means "loop to self" — there is no next block.  The boot ROM treats this as a single-block chain.

### Marker End

```
.word  0xab123579                                // PICOBIN_BLOCK_MARKER_END
```

The closing magic number.  The boot ROM matches the start and end markers to validate the block.

## The Complete Block in Memory

Starting at address 0x10000000 (beginning of flash):

```
  Offset  Hex           Description
  ------  ----------    -----------
  0x00    d3 de ff ff   PICOBIN_BLOCK_MARKER_START
  0x04    42            IMAGE_TYPE tag (1BS, kind=2)
  0x05    01            size: 1 word
  0x06    21 10         SECURE ARM EXE (0x1021)
  0x08    ff            LAST item tag
  0x09    01 00         size: 1 word
  0x0b    00            pad
  0x0c    00 00 00 00   next block pointer (self)
  0x10    79 35 12 ab   PICOBIN_BLOCK_MARKER_END
```

Total size: 20 bytes (0x14).  The vector table follows at offset 0x80 (128-byte aligned).

## Why Secure Mode?

The RP2350 supports Secure and Non-Secure execution modes (ARM TrustZone).  Our firmware boots in Secure mode, which gives it full access to all peripherals and memory regions without SAU (Security Attribution Unit) restrictions.  If we used Non-Secure mode, certain registers would be inaccessible.

## Summary

The IMAGE_DEF block is a 20-byte metadata structure that the boot ROM requires.  It contains start and end markers, an image type descriptor (ARM Secure EXE), a last-item terminator, and a self-referencing next-block pointer.  Every byte has a specific meaning defined by the PICOBIN format.  Without this block, the RP2350 will not boot from flash.
