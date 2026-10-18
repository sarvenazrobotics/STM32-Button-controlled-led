# STM32 Button-Controlled LED

An STM32CubeMX-generated firmware demonstrating **short-press** and **long-press** button detection on a single GPIO-driven LED.

- **Short press (< 3 s)** → LED blinks for 1 second
- **Long press (≥ 3 s)** → LED turns on while holding and stays on after release

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Behavior](#behavior)
- [Proteus Simulation](#Proteus-Simulation)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Project Structure](#project-structure)
- [Pinout](#pinout)
- [Building the Project](#building-the-project)
- [Flashing the Firmware](#flashing-the-firmware)
- [How It Works](#how-it-works)
- [Configuration Notes](#configuration-notes)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [License](#license)

---

## Overview

This project is a minimal, bare-metal firmware example generated with **STM32CubeMX** and built with **STM32CubeIDE**. It exercises two core concepts every embedded beginner should master:

1. **Reading digital inputs** with debouncing (button press / release)
2. **Driving digital outputs** with well-defined timing semantics (short vs. long press)

The firmware is intentionally kept small and side-effect free beyond what is documented here, so it can serve as a template for larger projects.

---

## Features

- ✅ Debounced button input (20 ms press, 50 ms release)
- ✅ Distinguishes between a **short press** and a **long press** (≥ 3 s)
- ✅ Immediate visual feedback: the LED turns on at the exact moment the 3-second threshold is crossed, without waiting for the button release
- ✅ No dynamic memory, no RTOS, no external libraries beyond the ST HAL
- ✅ Compiler-safe polling loop (works even with `-Os` / `-O2`)
- ✅ Non-blocking short-press handling (1 s blink after release)

---

## Behavior

| User Action                  | LED Behavior                                                   |
| ---------------------------- | -------------------------------------------------------------- |
| Tap and release (< 3 s)      | LED turns **ON** for 1 second, then turns **OFF**              |
| Hold for ≥ 3 s, then release | LED turns **ON** at the 3-second mark and **stays ON**         |
| No interaction               | LED stays in whatever state was set by the last interaction    |

> 💡 Starting behavior: `LED1(0)` runs once at boot, so the LED is **OFF** at power-up.

---

### Proteus Simulation

https://github.com/user-attachments/assets/bfe00aa3-44a1-4c0a-866d-9a629275a66e



## Hardware Requirements

- An STM32 development board supported by the current CubeMX project
  (clock config uses **HSI**, no PLL, `FLASH_LATENCY_0` — a low-speed part such as **STM32F0 / F1 / F3 / L0 / L4** is expected)
- One **push button** wired between `KEY1_Pin` and **GND** (internal pull-up is enabled in firmware)
- One **LED** wired between `LED1_Pin` and **GND** (assuming active-low logic — see [Configuration Notes](#configuration-notes))
- ST-Link (on-board or external) for flashing / debugging

---

## Software Requirements

| Tool               | Version (recommended) |
| ------------------ | --------------------- |
| STM32CubeIDE       | 1.14 or newer         |
| STM32CubeMX        | 6.10 or newer         |
| STM32Cube FW (HAL) | matching the MCU      |
| GCC ARM toolchain  | bundled with CubeIDE  |

---

## Project Structure

```
.
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       ├── main.c
│       ├── stm32*_hal_msp.c
│       ├── stm32*_it.c
│       └── system_stm32*.c
├── Drivers/
│   ├── CMSIS/
│   └── STM32*_HAL_Driver/
├── *.ioc                       # STM32CubeMX project file
├── STM32*.ld                   # Linker script
├── startup_stm32*.s            # Startup code
├── .project / .cproject        # Eclipse project metadata
└── README.md                   # ← you are here
```

> The `.ioc` file is the **source of truth** for pinout and clock configuration — always regenerate code from it when changing hardware settings.

---

## Pinout

| Signal | Port / Pin   | Mode         | Pull    | Active Level                        |
| ------ | ------------ | ------------ | ------- | ----------------------------------- |
| LED1   | *(see .ioc)* | Output PP    | None    | **Low** (`GPIO_PIN_RESET` = ON)     |
| KEY1   | *(see .ioc)* | Input        | Pull-up | **Low** (`GPIO_PIN_RESET` = pressed) |

Exact port/pin numbers are defined in `main.h` via `LED1_Pin`, `LED1_GPIO_Port`, `KEY1_Pin`, `KEY1_GPIO_Port`. Adjust them in CubeMX if your board differs.

---

## Building the Project

### In STM32CubeIDE

1. Open the IDE, choose **File ▸ Open Projects from File System…** and select this repository.
2. Wait for the workspace to index (`Drivers/` and `.ioc` linkage are auto-resolved).
3. Select the **Debug** build configuration (recommended for first use).
4. **Project ▸ Build Project** (`Ctrl+B`).

The resulting artifact is written to `Debug/<project>.elf`.

### From the command line (Makefile)

If your CubeMX configuration exported a `Makefile`:

```bash
make -j$(nproc)
```

---

## Flashing the Firmware

### STM32CubeIDE

- **Run ▸ Debug** (`F11`) — launches the debugger, flashes, and halts at `main()`.
- **Run ▸ Run** (`Ctrl+F11`) — flashes and runs without halting.

### STM32CubeProgrammer (CLI)

```bash
STM32_Programmer_CLI -c port=SWD -w Debug/<project>.elf -rst
```

### OpenOCD

```bash
openocd -f interface/stlink.cfg -f target/<mcu>.cfg \
        -c "program Debug/<project>.elf verify reset exit"
```

---

## How It Works

The entire application logic lives in the `while(1)` loop of `main.c` (inside the `USER CODE BEGIN 3` block). It follows a classic **state-based press detector**:

### Step 1 — Press detection with debounce

```c
if (KEY1 == GPIO_PIN_RESET) {
    HAL_Delay(20);                       // ignore contact bounce
    if (KEY1 == GPIO_PIN_RESET) {        // confirmed press
```

### Step 2 — Wait for release, track duration

```c
uint32_t start_time = HAL_GetTick();
uint8_t  long_press = 0;

while (KEY1 == GPIO_PIN_RESET) {
    if (!long_press && (HAL_GetTick() - start_time >= 3000)) {
        long_press = 1;
        led_state  = 1;
        LED1(led_state);                 // immediate feedback at 3 s
    }
}
```

Two details make this robust:

- The **`if` inside the loop** prevents the compiler from optimizing away the empty wait (a known pitfall when using `-Os` / `-O2`).
- The LED turns **ON the moment** the 3-second threshold is reached — the user does not have to release the button to see the effect.

### Step 3 — Release debounce and post-release action

```c
HAL_Delay(50);                            // absorb release bounce

if (!long_press) {
    LED1(1);                              // short press → blink
    HAL_Delay(1000);
    LED1(led_state);                      // restore previous state
}
```

### Step 4 — Idle back-off

```c
HAL_Delay(10);                            // yield CPU when idle
```

---

## Configuration Notes

### LED polarity

The macro assumes an **active-low** LED (pin driven low = ON):

```c
#define LED1(x) ((x==1) ? HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET) \
                        : HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET))
```

If your LED is active-high, swap `GPIO_PIN_RESET` and `GPIO_PIN_SET` in this macro.

### Button polarity

`KEY1_Pin` is configured as `GPIO_MODE_INPUT` with `GPIO_PULLUP`, so:

- **Not pressed** → `GPIO_PIN_SET` (HIGH)
- **Pressed**     → `GPIO_PIN_RESET` (LOW)

If you use an external pull-down or active-high button, invert the comparison in the two `if (KEY1 == GPIO_PIN_RESET)` checks.

### Timing constants

| Constant     | Location                   | Meaning                       |
| ------------ | -------------------------- | ----------------------------- |
| `20 ms`      | press debounce             | contact settling              |
| `50 ms`      | release debounce           | absorb contact bounce         |
| `3000 ms`    | long-press threshold       | `HAL_GetTick() - start_time`  |
| `1000 ms`    | short-press blink duration | `HAL_Delay` after release     |
| `10 ms`      | idle back-off              | reduces CPU / power           |

All constants are hard-coded in `main.c`. Convert them to `#define`s if you plan to expose them via a header.

---



## Troubleshooting

| Symptom                                            | Likely Cause / Fix                                                                  |
| -------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Short press works, long press never stays          | Empty wait loop optimized away — verify the `if` inside the `while` loop is present |
| LED behaves inverted                               | Active-low vs. active-high mismatch — swap `GPIO_PIN_SET` / `GPIO_PIN_RESET` in `LED1(x)` |
| Every long press is detected as many short presses | Release debounce missing — ensure `HAL_Delay(50)` runs after the wait loop          |
| Button never detected                              | Wrong pull-up — confirm `GPIO_PULLUP` is set in `MX_GPIO_Init`                      |
| Nothing happens at all                             | Verify `LED1_Pin` / `KEY1_Pin` in `main.h` match your board                         |
| `HAL_Delay` hangs forever                          | SysTick not running — check `HAL_Init()` was called before `SystemClock_Config()`   |

---

## Known Limitations

- **Blocking design** — `HAL_Delay()` is used throughout; the CPU is fully occupied during the 1 s blink and during button holding. For a production design, use TIM-based debouncing and an interrupt-driven state machine.
- **No toggle on long press** — a long press *always* turns the LED ON. To toggle instead, replace the long-press branch with:
  ```c
  led_state = !led_state;
  LED1(led_state);
  ```
- **No low-power mode** — the idle `HAL_Delay(10)` keeps the CPU awake. Consider `__WFI()` or `HAL_PWR_EnterSLEEPMode()` for battery-powered applications.
- **Optimization sensitivity** — the code works correctly at any optimization level, but you should still verify behavior after changing compiler flags.

---

## License

Unless otherwise stated, this project is derived from STMicroelectronics HAL code, which is licensed under the **BSD-3-Clause** license as distributed in `Drivers/CMSIS/LICENSE.txt` and `Drivers/STM32*_HAL_Driver/LICENSE.txt`.



---

