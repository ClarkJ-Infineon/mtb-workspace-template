---
name: peripheral-init
description: Initialize peripherals on PSOC Edge E84 — GPIO, UART, I2C, SPI, Timer, PWM, ADC, SysInt, and more. Provides init sequences, Device Configurator config struct names, BSP mappings, and common gotchas. Use when a user asks how to initialize a peripheral or when peripheral init appears to fail silently.
---

# Peripheral Init Quick Reference — PSOC Edge E84

Use this skill when a user needs to initialize a peripheral on PSOC Edge E84, or when a peripheral appears to be non-functional after initialization.

> **Full reference:** Read `reference/peripheral-init-reference.md` for complete init sequences, code examples, and gotcha tables for all peripherals.

---

## Quick Diagnosis — "Peripheral Appears Dead"

Before diving into per-peripheral details, check these universal causes:

| Symptom | Most Likely Cause | Fix |
|---------|-------------------|-----|
| UART produces no output | Missing `Cy_SCB_UART_Enable()` call after Init | Add `Cy_SCB_UART_Enable(base)` |
| I2C/SPI transfer hangs | Missing ISR registration for async transfers | Register ISR with `Cy_SysInt_Init()` + `NVIC_EnableIRQ()` |
| Timer/PWM counter stuck at 0 | Missing `Cy_TCPWM_TriggerStart_Single()` | Add TriggerStart after Enable |
| ADC reads always 0 | Missing `Cy_AutAnalog_StartAutonomousControl()` | Add StartAutonomousControl after Init |
| GPIO write has no effect | Wrong HSIOM setting (pin routed to peripheral, not GPIO) | Set `hsiom = HSIOM_SEL_GPIO` or use DC |
| Peripheral works on one core but not the other | I2C/SPI bus ownership is per-core; no cross-core arbitration | Move all users of that bus to one core |

---

## Init Pattern — Every Peripheral Follows This

```
Init(base, &config, &context) → Enable(base) → Start/Trigger → Read/Write
```

**Bold steps below are commonly forgotten:**

| Peripheral | Init | **Enable** | **Start** | Use |
|---|---|---|---|---|
| GPIO | `Cy_GPIO_Pin_Init()` | — | — | `Cy_GPIO_Read/Write()` |
| UART | `Cy_SCB_UART_Init()` | **`Cy_SCB_UART_Enable()`** | — | `Cy_SCB_UART_Put/Get()` |
| I2C | `Cy_SCB_I2C_Init()` | **`Cy_SCB_I2C_Enable()`** | — | `Cy_SCB_I2C_MasterRead/Write()` |
| SPI | `Cy_SCB_SPI_Init()` | **`Cy_SCB_SPI_Enable()`** | — | `Cy_SCB_SPI_Transfer()` (+ ISR!) |
| Timer | `Cy_TCPWM_Counter_Init()` | `Counter_Enable()` | **`TriggerStart_Single()`** | `GetCounter()` |
| PWM | `Cy_TCPWM_PWM_Init()` | `PWM_Enable()` | **`TriggerStart_Single()`** | — |
| ADC | `Cy_AutAnalog_Init()` | — | **`StartAutonomousControl()`** | `SAR_ReadResult()` |
| SysInt | `Cy_SysInt_Init()` | `NVIC_EnableIRQ()` | — | — |

---

## PDL vs HAL Decision

**Recommendation for PSOC Edge:** Use PDL (`Cy_*` APIs). Device Configurator generates PDL config structs, and all official PSOC Edge code examples use PDL.

**HAL prefix on PSOC Edge is `mtb_hal_*`** — NOT `cyhal_*` (which is PSOC 6). This is a common porting mistake.

---

## BSP Peripheral Mapping (KIT_PSE84_EVAL_EPC2)

| BSP Define | HW Instance | Function |
|---|---|---|
| `CYBSP_I2C_CONTROLLER_HW` | SCB0 | I2C (sensors, touch) |
| `CYBSP_DEBUG_UART_HW` | SCB2 | Debug UART |
| `CYBSP_BT_UART_HW` | SCB4 | Bluetooth HCI |
| `CYBSP_EZ_I2C_TARGET_HW` | SCB5 | EZ-I2C (KitProg3) |
| `CYBSP_SPI_CONTROLLER_HW` | SCB10 | SPI (radar, flash) |
| `CYBSP_GENERAL_PURPOSE_TIMER_HW` | TCPWM0[0] | Timer |
| `CYBSP_PWM_LED_CTRL_HW` | TCPWM0[5] | PWM LED |
| `CYBSP_WIFI_SDIO_HW` | SDHC0 | WiFi SDIO |
| `CYBSP_AUTONOMOUS_ANALOG` | AUTANALOG | ADC |

> **For full init code, gotcha tables, and Tier 2/3 peripherals**, read `reference/peripheral-init-reference.md`.
