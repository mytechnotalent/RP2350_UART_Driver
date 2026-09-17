## FREE Embedded Hacking Course [HERE](https://github.com/mytechnotalent/Embedded-Hacking)
### VIDEO PROMO [HERE](https://www.youtube.com/watch?v=aD7X9sXirF8)

<br>

# RP2350 UART Driver
An RP2350 UART driver written entirely in ARM Assembler.

<br>

# Install ARM Toolchain (Windows / RP2350 Cortex-M33)
Official Raspberry Pi guidance for RP2350 ARM recommends the Arm GNU Toolchain from developer.arm.com.

## Official References
- Raspberry Pi Pico SDK quick start: [HERE](https://github.com/raspberrypi/pico-sdk#quick-start-your-own-project)
- Tool downloads (official): [HERE](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)

## Install (PowerShell)
```powershell
$url = "https://developer.arm.com/-/media/Files/downloads/gnu/15.2.rel1/binrel/arm-gnu-toolchain-15.2.rel1-mingw-w64-x86_64-arm-none-eabi.zip"
$zipPath = "$env:TEMP\arm-toolchain-15-x64-win.zip"
$extractPath = "$env:TEMP\arm-extract"
$dest = "$HOME\arm-toolchain-15"

Invoke-WebRequest -Uri $url -OutFile $zipPath
Expand-Archive -LiteralPath $zipPath -DestinationPath $extractPath -Force
Move-Item "$extractPath\arm-gnu-toolchain-*" $dest -Force
Get-ChildItem -Path $dest | Select-Object Name
```

## Add Toolchain To User PATH (PowerShell)
```powershell
$toolBin = "$HOME\arm-toolchain-15\bin"
$currentUserPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($currentUserPath -notlike "*$toolBin*") {
  [Environment]::SetEnvironmentVariable("Path", "$currentUserPath;$toolBin", "User")
}
```

Close and reopen your terminal after updating PATH.

## Verify Toolchain
```powershell
arm-none-eabi-as --version
arm-none-eabi-ld --version
arm-none-eabi-objcopy --version
```

## Build This Project
```powershell
.\build.bat
```

## UART Terminal Setup (PuTTY)
- Speed: 115200
- Data bits: 8
- Stop bits: 1
- Parity: None
- Flow control: None

## UART Wiring (Pico 2 Target)
- GP0 = UART0 TX (target output)
- GP1 = UART0 RX (target input)
- GND must be common between target and USB-UART adapter/debug probe
- Cross wiring is required: adapter TX -> GP1, adapter RX -> GP0

<br>

# Build
```
.\build.bat
```

<br>

# Clean
```
.\clean.bat
```

<br>

# main.s Code
```
/**
 * FILE: main.s
 *
 * DESCRIPTION:
 * RP2350 Bare-Metal UART Main Application.
 * 
 * BRIEF:
 * Main application entry point for RP2350 UART driver. Contains the
 * main loop that echoes UART input to output.
 *
 * AUTHOR: Kevin Thomas
 * CREATION DATE: November 2, 2025
 * UPDATE DATE: November 27, 2025
 */

.syntax unified                                  // use unified assembly syntax
.cpu cortex-m33                                  // target Cortex-M33 core
.thumb                                           // use Thumb instruction set

.include "constants.s"

/**
 * Initialize the .text section. 
 * The .text section contains executable code.
 */
.section .text                                   // code section
.align 2                                         // align to 4-byte boundary

/**
 * @brief   Main application entry point.
 *
 * @details Implements the infinite blink loop.
 *
 * @param   None
 * @retval  None
 */
.global main                                     // export main
.type main, %function                            // mark as function
main:
.Push_Registers:
  push  {r4-r12, lr}                             // push registers r4-r12, lr to the stack
.Loop:
  bl    UART0_In                                 // call UART0_In
  bl    UART0_Out                                // call UART0_Out
  b     .Loop                                    // loop forever
.Pop_Registers:
  pop   {r4-r12, lr}                             // pop registers r4-r12, lr from the stack
  bx    lr                                       // return to caller

/**
 * Test data and constants.
 * The .rodata section is used for constants and static data.
 */
.section .rodata                                 // read-only data section

/**
 * Initialized global data.
 * The .data section is used for initialized global or static variables.
 */
.section .data                                   // data section

/**
 * Uninitialized global data.
 * The .bss section is used for uninitialized global or static variables.
 */
.section .bss                                    // BSS section
```

<br>

# License
[Apache License 2.0](https://github.com/mytechnotalent/RP2350_UART_Driver/blob/main/LICENSE)
