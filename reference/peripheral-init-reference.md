# PSOC Edge E84 — Peripheral Init Quick Reference

> **Target device:** PSOC Edge PSE84 (KIT_PSE84_EVAL_EPC2 / KIT_PSE84_AI)
> **DSL version:** mtb-dsl-pse8xxgp (see CONTEXT.md for current version)
> **Audience:** AI agents and developers initializing peripherals on PSOC Edge E84
> **Companion docs:** `psoc-edge-dual-core-guide.md` (multi-core patterns), `psoc-edge-porting-guide.md` (PSOC 6 → Edge migration)

---

## How to Use This Reference

**For AI agents:** When a developer asks "how do I init the ADC?" or "my UART isn't working," look up the peripheral below. The init sequence table tells you the exact call order; the gotcha annotations tell you what fails silently.

**For developers:** Jump to your peripheral. Check the init sequence, verify you haven't missed a step (bold = commonly forgotten), and review the gotchas.

**When to use domain agents instead:**
- Adding WiFi, MQTT, BLE → use `#peripheral-init` skill for the peripheral layer, then `wifi-mqtt` or `ble-setup` skills for the protocol layer
- Adding LVGL display → use `lvgl-setup` skill
- Setting up dual-core IPC → use `dual-core-setup` skill
- Diagnosing build failures or HardFaults → use `build-error` skill

This reference covers **individual peripheral initialization** — the building blocks those agents orchestrate.

---

## PDL vs HAL Decision Guide

| Criteria | Use HAL (`mtb_hal_*`) | Use PDL (`Cy_*`) |
|----------|:---:|:---:|
| Portability across PSOC families | ✅ | ❌ |
| Simple peripheral init/use | ✅ | ✅ |
| Advanced peripheral configuration | ❌ | ✅ |
| Register-level control needed | ❌ | ✅ |
| Peripheral not available in HAL | N/A | ✅ (only option) |
| Used by Device Configurator generated code | — | ✅ (DC generates PDL config structs) |

**PSOC Edge recommendation:** Use PDL for most work. Device Configurator generates PDL config structs (e.g., `CYBSP_DEBUG_UART_config`), and most PSOC Edge code examples use PDL directly. HAL is useful for portability-conscious code targeting both PSOC Edge and PSOC 6.

**⚠️ PSOC Edge HAL prefix is `mtb_hal_*`** — not `cyhal_*` (which is PSOC 6). See lessons-learned §6.3.

---

## Device Configurator vs. Manual Init

Most peripherals on PSOC Edge can be configured two ways:

| Method | When to Use | How It Works |
|--------|-------------|--------------|
| **Device Configurator (DC)** | Standard peripherals with GUI-configurable parameters (UART, I2C, SPI, Timer, ADC) | Configure in DC GUI → generates `cycfg_peripherals.h/.c` with config structs → call `Cy_*_Init()` with generated struct |
| **Manual PDL init** | Dynamic configuration, peripherals not in DC, or runtime reconfiguration | Define config struct in code → call `Cy_*_Init()` directly |

**Rule:** If the peripheral has a DC personality, prefer DC configuration. The generated config structs are validated by DC's DRC engine. Manual init is error-prone for complex peripherals (clocks, pins, interrupts are auto-wired by DC).

---

## Init Sequence Master Table

Every peripheral follows the pattern: **Init → Configure → Enable → Start → Read/Write**. Skipping any step causes the peripheral to fail silently.

| Peripheral | Init | Configure | Enable | **Start** | Read/Write |
|---|---|---|---|---|---|
| **GPIO** | `Cy_GPIO_Pin_Init(port, pin, &cfg)` | — | — | — | `Cy_GPIO_Read/Write()` |
| **UART (SCB)** | `Cy_SCB_UART_Init(base, &cfg, &ctx)` | — | **`Cy_SCB_UART_Enable(base)`** | — | `Cy_SCB_UART_Put/Get()` |
| **I2C (SCB)** | `Cy_SCB_I2C_Init(base, &cfg, &ctx)` | Slave buffers | **`Cy_SCB_I2C_Enable(base)`** | — | `Cy_SCB_I2C_MasterRead/Write()` |
| **SPI (SCB)** | `Cy_SCB_SPI_Init(base, &cfg, &ctx)` | — | **`Cy_SCB_SPI_Enable(base)`** | — | `Cy_SCB_SPI_Transfer()` |
| **Timer (TCPWM)** | `Cy_TCPWM_Counter_Init(base, num, &cfg)` | — | `Cy_TCPWM_Counter_Enable(base, num)` | **`Cy_TCPWM_TriggerStart_Single()`** | `Cy_TCPWM_Counter_GetCounter()` |
| **PWM (TCPWM)** | `Cy_TCPWM_PWM_Init(base, num, &cfg)` | — | `Cy_TCPWM_PWM_Enable(base, num)` | **`Cy_TCPWM_TriggerStart_Single()`** | — |
| **ADC (AUTANALOG)** | `Cy_AutAnalog_Init(&cfg)` | `Cy_AutAnalog_SetInterruptMask()` | — | **`Cy_AutAnalog_StartAutonomousControl()`** | `Cy_AutAnalog_SAR_ReadResult()` |
| **SysInt** | `Cy_SysInt_Init(&cfg, handler)` | `NVIC_SetPriority()` | `NVIC_EnableIRQ()` | — | — |

**Bold = commonly forgotten step that causes "peripheral appears dead."**

---

## Tier 1 Peripheral Details

### GPIO

**Header:** `cy_gpio.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__gpio.html
**DC Personality:** `pin` — configure in Device Configurator Pins tab

#### Init Sequence

```c
#include "cy_gpio.h"
#include "cycfg_pins.h"  // DC-generated pin config

// Option 1: Use DC-generated config (preferred)
// DC generates Cy_GPIO_Pin_Init() calls in cycfg_pins.c — called by cybsp_init()

// Option 2: Manual init
cy_stc_gpio_pin_config_t pin_cfg = {
    .outVal    = 0,
    .driveMode = CY_GPIO_DM_STRONG_IN_OFF,  // Output push-pull
    .hsiom     = HSIOM_SEL_GPIO,             // GPIO function (not peripheral)
    .intEdge   = CY_GPIO_INTR_DISABLE,
    .vtrip     = CY_GPIO_VTRIP_CMOS,
};
Cy_GPIO_Pin_Init(GPIO_PRT5, 0, &pin_cfg);

// Read/Write
Cy_GPIO_Write(CYBSP_USER_LED_PORT, CYBSP_USER_LED_PIN, 1);  // Set HIGH
uint32_t val = Cy_GPIO_Read(GPIO_PRT5, 0);                   // Read pin
Cy_GPIO_Inv(CYBSP_USER_LED_PORT, CYBSP_USER_LED_PIN);        // Toggle
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **KIT_PSE84_AI USER LED is active-HIGH** — most other Infineon kits use active-low. `Write(1)` = ON. | Inverted blink logic |
| G2 | **HSIOM must match function** — setting `driveMode` without correct `hsiom` value means the pin is GPIO even if you think it's connected to a peripheral. DC handles this automatically. | Peripheral appears dead |
| G3 | **Interrupt edge vs. level** — `CY_GPIO_INTR_RISING` requires the pin to actually transition. A pin that's already HIGH won't trigger on init. | Missed first interrupt |

#### HAL Alternative

```c
#include "mtb_hal_gpio.h"
mtb_hal_gpio_init(CYBSP_USER_LED, MTB_HAL_GPIO_DIR_OUTPUT, MTB_HAL_GPIO_DRIVE_STRONG, 0);
mtb_hal_gpio_write(CYBSP_USER_LED, 1);
mtb_hal_gpio_toggle(CYBSP_USER_LED);
```

---

### UART (SCB)

**Header:** `cy_scb_uart.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__scb__uart.html
**DC Personality:** `uart` — configure in Device Configurator Peripherals tab
**BSP defines:** `CYBSP_DEBUG_UART_HW` = SCB2, `CYBSP_DEBUG_UART_IRQ` = scb_2_interrupt_IRQn

#### Init Sequence

```c
#include "cy_scb_uart.h"
#include "cycfg_peripherals.h"  // DC-generated UART config

// Context struct (required — stores runtime state)
cy_stc_scb_uart_context_t uart_context;

// 1. INIT — pass DC-generated config
cy_en_scb_uart_status_t status = Cy_SCB_UART_Init(
    CYBSP_DEBUG_UART_HW,        // SCB2
    &CYBSP_DEBUG_UART_config,   // DC-generated
    &uart_context
);
CY_ASSERT(status == CY_SCB_UART_SUCCESS);

// 2. ENABLE — MUST come after Init (commonly forgotten!)
Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);

// 3. USE
Cy_SCB_UART_PutString(CYBSP_DEBUG_UART_HW, "Hello PSOC Edge\r\n");
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **`Enable()` MUST be called after `Init()`** — without it, UART silently drops all data. No error returned. | No serial output; looks like a hang |
| G2 | **retarget-io on PSOC Edge needs a wrapper** — the standard `cy_retarget_io_init()` doesn't exist on E84. Use the BSP-provided retarget-io wrapper or init UART manually and redirect `printf`. See lessons-learned §5.1. | printf produces no output |
| G3 | **stdout is line-buffered** — partial prints (no `\n`) never reach UART. Always end printf with `\n` or call `fflush(stdout)`. | Misdiagnoses hang location |
| G4 | **Default oversampling is 16** — baud rate = source_clock / (oversample × divider). If you calculate dividers manually, use 16, not 8. DC handles this automatically. | Wrong baud rate |
| G5 | **Context struct must persist** — `uart_context` is referenced by the driver at runtime. Declaring it as a local variable in a function that returns → use-after-free. | Intermittent crashes |

#### Reference CE
`mtb-example-psoc-edge-hello-world` — minimal UART init with retarget-io

---

### I2C (SCB)

**Header:** `cy_scb_i2c.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__scb__i2c.html
**DC Personality:** `i2c` — configure in Device Configurator Peripherals tab
**BSP defines:** `CYBSP_I2C_CONTROLLER_HW` = SCB0, `CYBSP_I2C_CONTROLLER_IRQ` = scb_0_interrupt_IRQn

#### Init Sequence

```c
#include "cy_scb_i2c.h"
#include "cycfg_peripherals.h"

cy_stc_scb_i2c_context_t i2c_context;

// 1. INIT
Cy_SCB_I2C_Init(CYBSP_I2C_CONTROLLER_HW, &CYBSP_I2C_CONTROLLER_config, &i2c_context);

// 2. ENABLE
Cy_SCB_I2C_Enable(CYBSP_I2C_CONTROLLER_HW);

// 3. USE — Master write example (blocking)
uint8_t write_buf[] = {0x00, 0x42};  // register addr + data
cy_en_scb_i2c_status_t result = Cy_SCB_I2C_MasterWrite(
    CYBSP_I2C_CONTROLLER_HW,
    &(cy_stc_scb_i2c_master_xfer_config_t){
        .slaveAddress = 0x76,          // BME280 default
        .buffer       = write_buf,
        .bufferSize   = sizeof(write_buf),
        .xferPending  = false
    },
    &i2c_context
);
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **I2C bus ownership is per-core** — arbitration across CM33/CM55 is NOT supported. One core must own the entire I2C bus. If CM55 runs the display + touch on SCB0, sensors on that bus must also move to CM55. | Bus corruption, random NAKs |
| G2 | **I2C clock divider affects speed** — DC sets the divider for Standard (100 kHz) or Fast (400 kHz) mode. Changing the HF clock without updating the divider gives wrong bus speed. For display touch controllers, ~184 kHz works best. | Device ID mismatches, comm failures |
| G3 | **Context struct must persist** — same as UART. Don't declare as local. | Use-after-free crashes |
| G4 | **Interrupt-driven I2C needs ISR registration** — if using `Cy_SCB_I2C_MasterRead()` in interrupt mode, register the ISR first (see SysInt section). Blocking mode works without ISR. | Hang on first transfer |

#### Reference CE
`mtb-example-psoc-edge-i2c-controller` — basic I2C controller with sensor read

---

### SPI (SCB)

**Header:** `cy_scb_spi.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__scb__spi.html
**DC Personality:** `spi` — configure in Device Configurator Peripherals tab
**BSP defines:** `CYBSP_SPI_CONTROLLER_HW` = SCB10, `CYBSP_SPI_CONTROLLER_IRQ` = scb_10_interrupt_IRQn

#### Init Sequence

```c
#include "cy_scb_spi.h"
#include "cycfg_peripherals.h"

cy_stc_scb_spi_context_t spi_context;

// 1. INIT
Cy_SCB_SPI_Init(CYBSP_SPI_CONTROLLER_HW, &CYBSP_SPI_CONTROLLER_config, &spi_context);

// 2. ENABLE
Cy_SCB_SPI_Enable(CYBSP_SPI_CONTROLLER_HW);

// 3. REGISTER ISR (required for Cy_SCB_SPI_Transfer!)
cy_stc_sysint_t spi_irq_cfg = {
    .intrSrc     = CYBSP_SPI_CONTROLLER_IRQ,
    .intrPriority = 3
};
Cy_SysInt_Init(&spi_irq_cfg, spi_interrupt_handler);
NVIC_EnableIRQ(CYBSP_SPI_CONTROLLER_IRQ);

// 4. USE
uint8_t tx_buf[16], rx_buf[16];
Cy_SCB_SPI_Transfer(CYBSP_SPI_CONTROLLER_HW, tx_buf, rx_buf, 16, &spi_context);
// Wait for completion
while (Cy_SCB_SPI_GetTransferStatus(CYBSP_SPI_CONTROLLER_HW, &spi_context)
       & CY_SCB_SPI_TRANSFER_ACTIVE) {}

// ISR handler
void spi_interrupt_handler(void) {
    Cy_SCB_SPI_Interrupt(CYBSP_SPI_CONTROLLER_HW, &spi_context);
}
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **`Cy_SCB_SPI_Transfer()` is interrupt-driven** — without a registered ISR, it hangs forever. This is the #1 SPI failure mode. | Infinite hang at first transfer |
| G2 | **Polling alternative exists** — for low-throughput use cases (e.g., radar sensor at 10 Hz), use `Cy_SCB_SPI_Write()` / `Cy_SCB_SPI_Read()` byte-by-byte. Simpler, no ISR needed. | — |
| G3 | **CS pin management** — DC can configure automatic CS assertion, or you can manage it manually via GPIO. If using manual CS, assert BEFORE transfer and deassert AFTER. | Slave doesn't respond |

#### Reference CE
`mtb-example-psoc-edge-spi-controller` — SPI controller with interrupt-driven transfer

---

### Timer / Counter (TCPWM)

**Header:** `cy_tcpwm_counter.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__tcpwm.html
**DC Personality:** `counter` — configure in Device Configurator Peripherals tab
**BSP defines:** `CYBSP_GENERAL_PURPOSE_TIMER_HW` = TCPWM0, `CYBSP_GENERAL_PURPOSE_TIMER_NUM` = 0

#### Init Sequence

```c
#include "cy_tcpwm_counter.h"
#include "cycfg_peripherals.h"

// 1. INIT
Cy_TCPWM_Counter_Init(CYBSP_GENERAL_PURPOSE_TIMER_HW,
                       CYBSP_GENERAL_PURPOSE_TIMER_NUM,
                       &CYBSP_GENERAL_PURPOSE_TIMER_config);

// 2. ENABLE
Cy_TCPWM_Counter_Enable(CYBSP_GENERAL_PURPOSE_TIMER_HW,
                         CYBSP_GENERAL_PURPOSE_TIMER_NUM);

// 3. START — commonly forgotten!
Cy_TCPWM_TriggerStart_Single(CYBSP_GENERAL_PURPOSE_TIMER_HW,
                              CYBSP_GENERAL_PURPOSE_TIMER_NUM);

// 4. READ
uint32_t count = Cy_TCPWM_Counter_GetCounter(CYBSP_GENERAL_PURPOSE_TIMER_HW,
                                              CYBSP_GENERAL_PURPOSE_TIMER_NUM);

// Optional: interrupt on terminal count
Cy_TCPWM_SetInterruptMask(CYBSP_GENERAL_PURPOSE_TIMER_HW,
                           CYBSP_GENERAL_PURPOSE_TIMER_NUM,
                           CY_TCPWM_INT_ON_TC);
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **`TriggerStart_Single()` is required after Enable** — Enable just powers the counter; it doesn't start counting. Without TriggerStart, `GetCounter()` always returns 0. | Counter appears dead |
| G2 | **TCPWM0 has 512 counters** — the `NUM` parameter is the counter index within the TCPWM block. DC assigns these; don't guess. Check `CYBSP_*_NUM` defines. | Wrong counter, corrupts another peripheral |
| G3 | **Clock source must be running** — TCPWM counters are clocked from a peripheral clock divider. If the divider isn't enabled (DC handles this), the counter doesn't increment. | Counter stuck at 0 |

---

### PWM (TCPWM)

**Header:** `cy_tcpwm_pwm.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__tcpwm.html
**DC Personality:** `pwm` — configure in Device Configurator Peripherals tab
**BSP defines:** `CYBSP_PWM_LED_CTRL_HW` = TCPWM0, `CYBSP_PWM_LED_CTRL_NUM` = 5

#### Init Sequence

```c
#include "cy_tcpwm_pwm.h"
#include "cycfg_peripherals.h"

// 1. INIT
Cy_TCPWM_PWM_Init(CYBSP_PWM_LED_CTRL_HW,
                   CYBSP_PWM_LED_CTRL_NUM,
                   &CYBSP_PWM_LED_CTRL_config);

// 2. ENABLE
Cy_TCPWM_PWM_Enable(CYBSP_PWM_LED_CTRL_HW,
                     CYBSP_PWM_LED_CTRL_NUM);

// 3. START — commonly forgotten!
Cy_TCPWM_TriggerStart_Single(CYBSP_PWM_LED_CTRL_HW,
                              CYBSP_PWM_LED_CTRL_NUM);

// 4. Adjust duty cycle at runtime
Cy_TCPWM_PWM_SetCompare0Val(CYBSP_PWM_LED_CTRL_HW,
                             CYBSP_PWM_LED_CTRL_NUM,
                             new_compare_value);
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **Same as Timer: `TriggerStart_Single()` required after Enable** | PWM output stuck at initial level |
| G2 | **PWM output pin must have correct HSIOM** — the pin must be routed to the TCPWM output via HSIOM, not left as GPIO. DC configures this; manual init must set the correct `hsiom` value. | Pin stays constant |
| G3 | **Dead-time PWM (`CYBSP_DEAD_TIME_PWM`)** — available on TCPWM0 counter 7. Useful for motor drive; has separate compare registers for rising/falling edges. | — |

---

### ADC (AUTANALOG — Autonomous Analog)

**Header:** `cy_autanalog.h` (umbrella), `cy_autanalog_sar.h` (SAR functions)
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__autanalog.html
**DC Personality:** `autanalog` — configure in Device Configurator Analog tab
**BSP defines:** `CYBSP_AUTONOMOUS_ANALOG_ENABLED`, `CYBSP_SAR_ADC_ENABLED`

> **⚠️ PSOC Edge ADC is fundamentally different from PSOC 6.** PSOC 6 uses `Cy_SAR_*` APIs. PSOC Edge uses the Autonomous Analog Controller (`Cy_AutAnalog_*`) which manages SAR, sequencer, and state machine as one unit.

#### Init Sequence

```c
#include "cy_autanalog.h"
#include "cycfg_peripherals.h"

// 1. INIT — pass DC-generated config (autonomous_analog_init is from DC)
Cy_AutAnalog_Init(&autonomous_analog_init);

// 2. CONFIGURE — set interrupt mask for SAR result ready
Cy_AutAnalog_SetInterruptMask(CY_AUTANALOG_INT_SAR0_RESULT);

// 3. START — begin autonomous ADC scanning (CRITICAL — commonly forgotten!)
Cy_AutAnalog_StartAutonomousControl();

// 4. READ — ADC values are now available
int32_t counts = Cy_AutAnalog_SAR_ReadResult(
    0,                          // SAR instance
    CY_AUTANALOG_SAR_INPUT_GPIO, // Input source
    0                           // Channel index
);

// 5. CONVERT to millivolts
int16_t mv = Cy_AutAnalog_SAR_CountsTo_mVolts(
    0,                          // SAR instance
    false,                      // Single-ended
    0,                          // Channel
    CY_AUTANALOG_SAR_INPUT_GPIO,
    0,                          // Sub-channel
    1800U,                      // Reference mV (VDDA/2 or Vref)
    counts
);
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **`StartAutonomousControl()` is the most commonly forgotten step** — Init alone does NOT begin conversion. Without it, `ReadResult()` returns 0 or stale data. | ADC reads always 0 |
| G2 | **DC configuration is essential** — the Autonomous Analog Controller has complex state machine configuration (scan groups, autonomous states). Manual configuration is extremely error-prone. Always use Device Configurator. | Misconfigured scan sequence |
| G3 | **GPIO channel vs. other inputs** — the SAR can read GPIO pins, internal references, or die temperature. The channel index maps to the scan group configuration in DC. | Reading wrong input |
| G4 | **Reference voltage** — `CountsTo_mVolts()` requires the correct reference voltage. On EPC2, the SAR reference is typically VDDA/2 (900 mV) or an external Vref. Check the DC configuration. | Incorrect voltage readings |

#### Reference CE
`mtb-example-psoc-edge-adc-basic` — basic ADC reading with Autonomous Analog Controller

---

### System Interrupts (SysInt)

**Header:** `cy_sysint.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__sysint.html
**No DC personality** — interrupts are configured in code

> **Required by:** Any interrupt-driven peripheral (SPI Transfer, I2C async, Timer TC interrupt, GPIO edge interrupt, UART RX interrupt).

#### Init Sequence

```c
#include "cy_sysint.h"

// 1. Define interrupt config
cy_stc_sysint_t my_irq_cfg = {
    .intrSrc     = CYBSP_SPI_CONTROLLER_IRQ,  // Use BSP define for the IRQ number
    .intrPriority = 3                          // 0 = highest, 7 = lowest (3-bit on CM33)
};

// 2. INIT — registers the handler function
Cy_SysInt_Init(&my_irq_cfg, my_interrupt_handler);

// 3. ENABLE — in NVIC
NVIC_EnableIRQ(CYBSP_SPI_CONTROLLER_IRQ);

// ISR handler — keep short, set a flag or call the peripheral's Interrupt() function
void my_interrupt_handler(void) {
    Cy_SCB_SPI_Interrupt(CYBSP_SPI_CONTROLLER_HW, &spi_context);
}
```

#### Gotchas

| # | Gotcha | Impact |
|---|--------|--------|
| G1 | **Priority levels are 3-bit on CM33** (0–7) and **8-bit on CM55** (0–255). Using priority 128 on CM33 wraps to a different value. | Unexpected preemption behavior |
| G2 | **FreeRTOS interrupt priority ceiling** — interrupts that call FreeRTOS API (`xSemaphoreGiveFromISR`, `xQueueSendFromISR`) must have priority ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY` (typically 1). Priority 0 interrupts CANNOT use FreeRTOS APIs. | Kernel corruption, random crashes |
| G3 | **Interrupt source names** — use `CYBSP_*_IRQ` defines from `cycfg_peripherals.h`, not raw IRQ numbers. The BSP may remap interrupt sources between device revisions. | Wrong interrupt triggered |
| G4 | **Clear interrupt flag in handler** — most peripherals require explicitly clearing the interrupt flag inside the ISR. Calling the peripheral's `*_Interrupt()` function (e.g., `Cy_SCB_SPI_Interrupt()`) does this automatically. If writing a custom handler, call `Cy_*_ClearInterrupt()`. | ISR fires continuously |

#### Priority Cheat Sheet

| Use Case | Recommended Priority (CM33) |
|----------|:---:|
| DMA completion | 1 |
| Communication peripherals (UART, SPI, I2C) | 2–3 |
| Timer / periodic interrupts | 3–4 |
| GPIO button / user input | 5–6 |
| Background / low-urgency | 7 |

---

## Tier 2 Peripherals (Condensed)

### DMA (AXIDMAC)

**Header:** `cy_axidmac.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__axidmac.html

**Init sequence:** `Cy_AXIDMAC_Channel_Init()` → `Cy_AXIDMAC_Channel_SetDescriptor()` → `Cy_AXIDMAC_Channel_Enable()` → `Cy_AXIDMAC_Channel_SetSwTrigger()` (or peripheral trigger)

**Key gotcha:** PSOC Edge uses AXIDMAC, not DW (Datawire) like PSOC 6. The API prefix is `Cy_AXIDMAC_*`, not `Cy_DMA_*`. Descriptors must be in non-cacheable memory if D-Cache is enabled — otherwise DMA reads stale data.

---

### RTC (Real-Time Clock)

**Header:** `cy_rtc.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__rtc.html

**Init sequence:** `Cy_RTC_Init(&cfg)` → `Cy_RTC_SetDateAndTime(&dateTime)` → `Cy_RTC_GetDateAndTime(&dateTime)`

**Key gotcha:** RTC uses a 32.768 kHz WCO/ILO clock. If using ILO (default), accuracy is ±15%. For timestamping applications, verify WCO is populated on your board.

---

### Watchdog Timer (WDT / MCWDT)

**Header:** `cy_wdt.h` (WDT), `cy_mcwdt.h` (Multi-Counter WDT)
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__wdt.html

**Init sequence:** `Cy_WDT_Init()` → `Cy_WDT_Enable()` → periodically call `Cy_WDT_ClearWatchdog()` to prevent reset

**Key gotchas:**
- **Secure boot enables WDT** — if you don't service or disable it, the device resets ~2s after boot. Call `Cy_WDT_Disable()` after `cybsp_init()` during development.
- **MCWDT is used by FreeRTOS tickless idle** — `CYBSP_CM33_LPTIMER_0_HW` = MCWDT_STRUCT0. Don't reconfigure MCWDT_STRUCT0 if using FreeRTOS tickless idle, or you'll get a HardFault ~2s after boot.

---

### Power Management (SysPm)

**Header:** `cy_syspm.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__syspm.html

**Init sequence:** Register callbacks → call `Cy_SysPm_CpuEnterDeepSleep()` when ready

**Key gotchas:**
- **Callback registration order matters** — peripherals must register their sleep/wake callbacks. WiFi and BLE stacks register their own callbacks; if your application enters DeepSleep before the stack is ready, the radio fails to wake.
- **FreeRTOS tickless idle IS a sleep mechanism** — with `configUSE_TICKLESS_IDLE=2`, the idle task calls DeepSleep automatically. You don't need to call it explicitly.

---

### System Clocks (SysClk)

**Header:** `cy_sysclk.h`
**API Docs:** https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__sysclk.html

**Recommendation:** Always configure clocks via Device Configurator. Manual clock configuration on PSOC Edge is complex (3 DPLLs, 8 CLK_HF paths, multiple PERI groups with independent dividers).

**Key gotchas:**
- **Changing CLK_HF frequency affects all peripherals on that path** — UART baud rates, SPI speeds, and timer periods all derive from CLK_HF via peripheral clock dividers. DC recalculates all dividers automatically; manual changes require recalculating every affected peripheral.
- **CM55 clock is independent** — CM55 runs from CLK_HF0 (up to 400 MHz). CM33 runs from a different CLK_HF. Changing one doesn't affect the other.

---

## Tier 3 Peripherals (Pointers)

For these specialized peripherals, use the appropriate domain agent or refer to the API docs directly.

| Peripheral | Header | API Docs | Domain Agent |
|---|---|---|---|
| **GFXSS (Graphics)** | `cy_graphics.h` | [GFXSS](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__graphics.html) | `lvgl-setup` skill |
| **MIPI DSI** | `cy_mipidsi.h` | [MIPI DSI](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__mipidsi.html) | `lvgl-setup` skill |
| **IPC** | `cy_ipc_drv.h` / `cy_ipc_pipe.h` / `cy_ipc_sema.h` | [IPC](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__ipc.html) | `dual-core-setup` skill |
| **TDM / I2S (Audio)** | `cy_tdm.h` | [TDM](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__tdm.html) | — |
| **PDM-PCM (Mic)** | `cy_pdm_pcm_v2.h` | [PDM-PCM](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__pdm__pcm__v2.html) | — |
| **CAN FD** | `cy_canfd.h` | [CAN FD](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__canfd.html) | — |
| **SD Host** | `cy_sd_host.h` | [SD Host](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__sd__host.html) | — |
| **I3C** | `cy_i3c.h` | [I3C](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__cy__i3c.html) | — |
| **SMIF (QSPI)** | `cy_smif.h` | [SMIF](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__smif.html) | — |
| **RRAM** | `cy_rram.h` | [RRAM](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__rram.html) | — |
| **NNLite (ML Accel)** | `cy_nnlite.h` | [NNLite](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__nnlite.html) | — |
| **Crypto** | `cy_crypto.h` | [Crypto](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__crypto.html) | — |
| **Ethernet (EMAC)** | `cy_ethif.h` | [EMAC](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__ethif.html) | — |
| **Security (MPC/PPC)** | `cy_mpc.h` / `cy_ppc.h` / `cy_ms_ctl.h` | [MPC](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__mpc.html), [PPC](https://infineon.github.io/mtb-dsl-pse8xxgp/html/group__group__ppc.html) | — |

---

## Appendix: BSP Peripheral Mapping (KIT_PSE84_EVAL_EPC2)

Quick reference for which SCB/TCPWM instance maps to which BSP function:

| BSP Define | HW Instance | Function |
|---|---|---|
| `CYBSP_I2C_CONTROLLER` | SCB0 | I2C controller (sensors, PMIC, touch) |
| `CYBSP_DEBUG_UART` | SCB2 | Debug UART (115200 default) |
| `CYBSP_BT_UART` | SCB4 | Bluetooth HCI UART |
| `CYBSP_EZ_I2C_TARGET` | SCB5 | EZ-I2C target (KitProg3 bridge) |
| `CYBSP_SPI_CONTROLLER` | SCB10 | SPI controller (radar sensor, external flash) |
| `CYBSP_GENERAL_PURPOSE_TIMER` | TCPWM0[0] | General-purpose timer |
| `CYBSP_PWM_LED_CTRL` | TCPWM0[5] | PWM for LED brightness |
| `CYBSP_DEAD_TIME_PWM` | TCPWM0[7] | Dead-time PWM (motor drive) |
| `CYBSP_WIFI_SDIO` | SDHC0 | WiFi SDIO interface (CYW55500) |
| `CYBSP_AUTONOMOUS_ANALOG` | AUTANALOG | ADC (autonomous controller) |
| `CYBSP_I3C_CONTROLLER` | I3C_CORE | I3C controller |
| `CYBSP_PDM` | PDM0 | Digital microphone |
| `CYBSP_TDM_CONTROLLER_0` | TDM_STRUCT0 | Audio TDM/I2S |
| `GFXSS_HW` | GFXSS | Graphics subsystem (GPU + DC) |
