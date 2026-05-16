---
name: psoc6-to-edge-migration
description: Guides porting of PSOC 6 (cyhal-based) projects to PSOC Edge E84 (mtb_hal + PDL). Covers HAL API migration, init model changes, retarget-io, GPIO, I2C, ADC, PWM, Timer, project structure, and Makefile differences. Use when a developer has existing PSOC 6 code and wants to run it on PSOC Edge.
---

# PSOC 6 → PSOC Edge E84 Migration

Use this skill when a developer has existing ModusToolbox™ code written for PSOC 6 with `cyhal_*` APIs and needs to make it run on PSOC Edge E84.

This is a real port, not a board-target rename. PSOC Edge E84 uses a different HAL family, a different initialization model, and a multi-core project structure.

## §1 — When to Use This Skill

Trigger this skill when any of these are true:

- The user has PSOC 6 application code using `cyhal_*` APIs and wants to move it to PSOC Edge E84.
- The build fails on a PSE84 target because `cyhal.h`, `cyhal_i2c.h`, `cyhal_uart.h`, or other `cyhal_*.h` headers are missing.
- The build fails with cryptic `mtb-hal-cat1` PSE84 include errors such as missing `cyhal_triggers_pse84.h`.
- The code assumes runtime pin-based peripheral init and now must be reworked for Device Configurator-generated objects.
- A PSOC 6 library or community example must be reused on PSOC Edge E84.

Do not use this skill for PSOC 6-to-PSOC 6 work. Keep `cyhal_*` there.

## §2 — Migration Checklist (ordered steps)

1. **Change project structure** (single → multi-core CM33 + CM55)
2. **Update BSP target** (`TARGET_CY8CPROTO-062-4343W` → `TARGET_KIT_PSE84_EVAL_EPC2` or similar)
3. **Replace `#include "cyhal.h"`** → `#include "mtb_hal.h"` (or specific `mtb_hal_*.h`)
4. **Replace pin-based init** with Device Configurator + setup/configure pattern
5. **Update retarget-io initialization** (3-step wrapper)
6. **Replace GPIO calls** (PDL for simple I/O, HAL setup for middleware integration)
7. **Update peripheral API calls** (I2C, SPI, UART, ADC, PWM, Timer)
8. **Migrate interrupt setup** (`cyhal_*_enable_event` → `Cy_SysInt_Init` + NVIC)
9. **Update Makefile COMPONENTS and DEFINES** (including BT baud, WCM, CY_RTOS_AWARE)
10. **Fix boot sequence ordering** (cybsp_init → retarget_io → BLE early init → CM55 enable → __enable_irq)
11. **Check library compatibility** (`cyhal`-dependent libs need shims or alternatives)
12. **Build, fix remaining errors, flash**

Port in that order. If you skip Device Configurator work and only rename APIs, the build will usually fail later on missing generated config objects.

## §3 — The Fundamental Architecture Change

PSOC 6 and PSOC Edge E84 do not share a compatible HAL layer.

- **PSOC 6:** `cyhal_*` from `mtb-hal-cat1`
- **PSOC Edge E84:** `mtb_hal_*` from the PSE84 Device Support Library (`mtb-dsl-pse8xxgp`)

### Key point

There is **NO compatibility shim** between `cyhal_*` and `mtb_hal_*`.

A straight rename of the target board is not enough because the init model changes:

- PSOC 6 creates peripherals from pins at runtime.
- PSOC Edge E84 expects Device Configurator-generated peripheral objects, explicit PDL init, then HAL setup.

### Dead-code trap

`mtb-hal-cat1` contains dead `#elif PSE84` code that references files that do not exist. That produces misleading first errors, but it is **NOT** real PSE84 support.

If a PSE84 build reports missing files inside `mtb-hal-cat1`, treat that as proof that a `cyhal`-dependent library was pulled into a PSOC Edge build.

### PDL vs HAL — two valid migration paths

Official Infineon PSOC Edge code examples consistently use **pure PDL** for peripheral access (e.g., `Cy_TCPWM_PWM_Init`, `Cy_AutAnalog_SAR_ReadResult`, `Cy_SCB_I2C_MasterSendStart`). Custom/middleware-heavy projects (bike-computer, sensor-hub, wifi-ble) use **HAL wrappers** (`mtb_hal_pwm_*`, `mtb_hal_adc_*`, `mtb_hal_i2c_*`).

Both are correct. Choose based on your needs:

| Use PDL when | Use HAL when |
|---|---|
| Simple peripheral access | Middleware expects HAL objects (`mtb_hal_*_t*`) |
| Config is fixed in Device Configurator | Runtime parameter changes needed |
| Following official Infineon CEs | Porting complex PSOC 6 apps with many `cyhal_*` calls |
| Peripheral has no HAL wrapper | Filtered/async reads (ADC), callbacks (PWM/Timer) |

### Init model comparison

**PSOC 6 Pattern:**
```c
cyhal_i2c_t i2c_obj;
cyhal_i2c_cfg_t cfg = { .is_slave = false, .frequencyhal_hz = 400000 };
cyhal_i2c_init(&i2c_obj, CYBSP_I2C_SDA, CYBSP_I2C_SCL, NULL);
cyhal_i2c_configure(&i2c_obj, &cfg);
```

**PSOC Edge Pattern:**
```c
mtb_hal_i2c_t i2c_obj;
cy_stc_scb_i2c_context_t i2c_context;
Cy_SCB_I2C_Init(CYBSP_I2C_HW, &CYBSP_I2C_config, &i2c_context);
Cy_SCB_I2C_Enable(CYBSP_I2C_HW, &i2c_context);
mtb_hal_i2c_setup(&i2c_obj, &CYBSP_I2C_hal_config, &i2c_context, NULL);
```

### Mental model to keep

For PSOC Edge E84, think:

1. Configure peripheral in Device Configurator
2. Use generated PDL config and context
3. Call `Cy_*_Init(...)`
4. Call `mtb_hal_*_setup(...)`
5. Use the HAL object

Not:

1. Pick pins in code
2. Call `cyhal_*_init(...)`
3. Start using the peripheral

### Boot sequence ordering (validated across multiple CEs)

PSOC Edge has a specific init ordering that differs from PSOC 6. Many PSOC 6 projects call `cybsp_init()` then immediately enable interrupts and start work. On PSOC Edge, the ordering matters more:

```c
// Validated pattern from wifi-ble, bike-computer, sensor-hub
int main(void)
{
    cy_rslt_t result = cybsp_init();           // 1. BSP init (clocks, pins, power)
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    init_retarget_io();                         // 2. UART/printf (before any debug output)

    // 3. Early subsystem init (BLE stack, etc.) — BEFORE enabling IRQs
    // ble_stack_early_init();                  // if BLE is used

    // 4. Enable CM55 if dual-core (must happen before __enable_irq on some flows)
    // Cy_SysEnableCM55(MXCM55, CY_CM55_APP_BOOT_ADDR, BOOT_WAIT_US);

    __enable_irq();                             // 5. Enable interrupts LAST

    // 6. Create FreeRTOS tasks and start scheduler
    // vTaskStartScheduler();
}
```

Key difference from PSOC 6: `__enable_irq()` is often delayed until after BLE init and CM55 boot. Enabling interrupts too early can cause race conditions with multi-core startup or BLE stack initialization.

## §4 — Retarget-IO (printf)

**PSOC 6:** `cy_retarget_io_init(HW, TX, RX, baud)` — single call  
**PSOC Edge:** 3-step wrapper (PDL init → HAL setup → retarget-io init)

Use this full pattern:

```c
// retarget_io_init.h
#ifndef _RETARGET_IO_INIT_H_
#define _RETARGET_IO_INIT_H_
#include "cybsp.h"
#include "mtb_hal.h"
#include "cy_retarget_io.h"
void init_retarget_io(void);
#endif

// retarget_io_init.c
#include "retarget_io_init.h"
static cy_stc_scb_uart_context_t DEBUG_UART_context;
static mtb_hal_uart_t DEBUG_UART_hal_obj;

void init_retarget_io(void)
{
    Cy_SCB_UART_Init(CYBSP_DEBUG_UART_HW, &CYBSP_DEBUG_UART_config, &DEBUG_UART_context);
    Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);
    mtb_hal_uart_setup(&DEBUG_UART_hal_obj, &CYBSP_DEBUG_UART_hal_config, &DEBUG_UART_context, NULL);
    cy_retarget_io_init(&DEBUG_UART_hal_obj);
}
```

Practical notes:

- Call `init_retarget_io()` after `cybsp_init()`.
- Add `DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF` for cleaner terminal output.
- Do not try to preserve the PSOC 6 four-argument `cy_retarget_io_init(...)` call shape on PSOC Edge.

## §5 — GPIO

**PSOC 6:** `cyhal_gpio_init(pin, dir, drive, val)` + `cyhal_gpio_write(pin, val)`  
**PSOC Edge:** BSP Device Configurator handles pin config. Use PDL directly for simple I/O. Use HAL setup when middleware requires HAL GPIO objects.

### Simple I/O — PDL direct (most cases)

```c
#include "cy_pdl.h"
#include "cybsp.h"
// No init needed — Device Configurator configures pins
Cy_GPIO_Write(CYBSP_USER_LED1_PORT, CYBSP_USER_LED1_PIN, 0u); // LED ON (active-low)
```

### HSIOM override — when pin mux needs runtime change

Some pins need their HSIOM function overridden at runtime (e.g., switching a pin from a peripheral function to GPIO mode):

```c
// Force pin to GPIO function (validated — ble-led project)
Cy_GPIO_SetHSIOM(CYBSP_USER_LED3_PORT, CYBSP_USER_LED3_PIN, HSIOM_SEL_GPIO);
Cy_GPIO_Write(CYBSP_USER_LED3_PORT, CYBSP_USER_LED3_PIN, value);
```

### HAL GPIO objects — when middleware needs them

WiFi, BLE, and some sensor middleware expect `mtb_hal_gpio_t` objects instead of raw PORT/PIN pairs. Use `mtb_hal_gpio_setup()` for those cases:

```c
// Validated — wifi-ble project, wifi_manager.c
mtb_hal_gpio_setup(&wcm_config.wifi_wl_pin,
    CYBSP_WIFI_WL_REG_ON_PORT_NUM, CYBSP_WIFI_WL_REG_ON_PIN);
mtb_hal_gpio_write(&wcm_config.wifi_wl_pin, 0);  // Assert reset
Cy_SysLib_Delay(2);
mtb_hal_gpio_write(&wcm_config.wifi_wl_pin, 1);  // Release reset
```

### Guidance

- **Default to PDL** (`Cy_GPIO_Write/Read/Inv`) for simple application GPIO.
- **Use HAL setup** only when a middleware API requires an `mtb_hal_gpio_t*` parameter.
- **Use HSIOM override** when a pin needs to change function at runtime (rare, but real in BLE LED examples).
- Do not add `cyhal_gpio_init(...)` replacements unless a specific library absolutely requires its own wrapper layer.

## §6 — Peripheral API Quick Reference Table

| Peripheral | PSOC 6 (`cyhal_*`) | PSOC Edge (`mtb_hal_*`) | Breaking Changes |
|---|---|---|---|
| I2C init | `cyhal_i2c_init(obj, sda, scl, clk)` | DC + `mtb_hal_i2c_setup(obj, cfg, ctx, NULL)` | Completely different init model |
| I2C init (PDL) | (same as above) | DC + `Cy_SCB_I2C_Init(HW, cfg, ctx)` + `Cy_SCB_I2C_Enable(HW)` | Official CE uses pure PDL — no HAL I2C object |
| I2C write | `cyhal_i2c_master_write(...)` | `mtb_hal_i2c_controller_write(...)` | `master` → `controller` |
| I2C write (PDL) | (same as above) | `Cy_SCB_I2C_MasterSendStart()` + `MasterWriteByte()` + `MasterSendStop()` | Byte-wise PDL in official CEs |
| I2C mem_write | `cyhal_i2c_master_mem_write(...)` | ❌ Removed | Must do manual reg+data write |
| I2C config struct | `is_slave`, `frequencyhal_hz` | `is_target`, `frequency_hz`, + `address_mask`, `enable_address_callback` | New fields, renamed |
| I2C enable_event | 4 params (has `intr_priority`) | 3 params (no priority) | Priority param removed |
| SPI init | `cyhal_spi_init(obj, mosi, miso, sclk, ssel, ...)` | DC + `mtb_hal_spi_setup(obj, cfg, ctx)` | Different init model |
| UART init | `cyhal_uart_init(obj, tx, rx, clk, cfg)` | DC + `mtb_hal_uart_setup(obj, cfg, ctx, NULL)` | Different init model |
| UART read/write | `cyhal_uart_read(obj, data, len)` | `mtb_hal_uart_read(obj, data, len)` | Same signature |
| ADC init | `cyhal_adc_init(obj, pin, clk)` + `cyhal_adc_channel_init_diff(ch, adc, vplus, vminus, cfg)` | DC + `Cy_AutAnalog_Init()` + `mtb_hal_adc_setup(obj, cfg, clk, channels)` | Requires AutAnalog PDL init first |
| ADC init (PDL) | (same as above) | DC + `Cy_AutAnalog_Init()` + `Cy_AutAnalog_SetInterruptMask()` + `Cy_AutAnalog_StartAutonomousControl()` | Official CE uses pure PDL — no HAL ADC object |
| ADC read | `cyhal_adc_read(ch)` (returns int32_t counts) / `cyhal_adc_read_uv(ch)` (returns int32_t microvolts) | PDL: `Cy_AutAnalog_SAR_ReadResult()` → `CountsTo_mVolts()` (int16_t mV) / HAL: `mtb_hal_adc_read_u16(ch)` (uint16_t 0-65535) | ⚠️ Three different return formats |
| ADC free | `cyhal_adc_free(obj)` + `cyhal_adc_channel_free(ch)` | No free — lifecycle managed by BSP | Changed |
| PWM init | `cyhal_pwm_init(obj, pin, clk)` | DC + `mtb_hal_pwm_setup(obj, cfg, clk)` | Different init model |
| PWM init (PDL) | (same as above) | DC + `Cy_TCPWM_PWM_Init(HW, NUM, cfg)` + `Cy_TCPWM_PWM_Enable()` + `Cy_TCPWM_TriggerStart_Single()` | Official CE uses pure PDL — no HAL PWM object |
| PWM set duty | `cyhal_pwm_set_duty_cycle(obj, duty_pct, freq_hz)` | `mtb_hal_pwm_set_period(obj, period_us, pulse_width_us)` | ⚠️ API change: duty percentage → explicit period/pulse in µs |
| PWM start/stop | `cyhal_pwm_start(obj)` / `cyhal_pwm_stop(obj)` | `mtb_hal_pwm_start(obj)` / `mtb_hal_pwm_stop(obj)` | Same |
| PWM enable_event | 4 params (has `intr_priority`) | 3 params (no priority) | Priority param removed |
| Timer init | `cyhal_timer_init(obj, clk)` | DC + `mtb_hal_timer_setup(obj, cfg, clk)` | Different init model |
| Timer start/stop | `cyhal_timer_start(obj)` / `cyhal_timer_stop(obj)` | `mtb_hal_timer_start(obj)` / `mtb_hal_timer_stop(obj)` | Same |
| Timer configure | `cyhal_timer_configure(obj, cfg)` — sets period, direction, one-shot at runtime | Not available — configured via Device Configurator | ❌ Runtime config removed |
| Timer read | `cyhal_timer_read(obj)` | `mtb_hal_timer_read(obj)` | Same |
| Timer reset | `cyhal_timer_reset(obj)` | `mtb_hal_timer_reset(obj, start_value)` | Added start_value param |
| Timer enable_event | 4 params (has `intr_priority`) | 3 params (no priority) | Priority param removed |
| GPIO init | `cyhal_gpio_init(pin, dir, drive, val)` | Use PDL: `Cy_GPIO_Write(PORT, PIN, val)` | DC handles pin config; no HAL init needed |
| GPIO write | `cyhal_gpio_write(pin, val)` | `Cy_GPIO_Write(PORT, PIN, val)` or `mtb_hal_gpio_write(obj, val)` | PDL for simple I/O; HAL when middleware needs `mtb_hal_gpio_t*` |
| GPIO read | `cyhal_gpio_read(pin)` | `Cy_GPIO_Read(PORT, PIN)` | Different API |
| GPIO HSIOM | N/A (implicit) | `Cy_GPIO_SetHSIOM(PORT, PIN, HSIOM_SEL_GPIO)` | Override pin mux at runtime when needed |
| GPIO HAL setup | N/A | `mtb_hal_gpio_setup(obj, port_num, pin)` | Only for middleware requiring HAL GPIO objects |
| Interrupt setup | `cyhal_*_register_callback()` + `cyhal_*_enable_event()` | `Cy_SysInt_Init(&cfg, handler)` + `NVIC_EnableIRQ(irq)` | Explicit PDL + NVIC replaces cyhal event model |

## §7 — ADC (Detailed Pattern)

**PSOC 6:**
```c
cyhal_adc_t adc_obj;
cyhal_adc_channel_t adc_channel;
cyhal_adc_init(&adc_obj, P10_0, NULL);
cyhal_adc_channel_init_diff(&adc_channel, &adc_obj, P10_0, CYHAL_ADC_VNEG, NULL);
int32_t adc_mv = cyhal_adc_read(&adc_channel); // counts (int32_t)
// Or: int32_t adc_uv = cyhal_adc_read_uv(&adc_channel); // microvolts
```

**PSOC Edge — PDL path (validated — official Infineon adc-basic CE):**
```c
#include "cybsp.h"
#include "retarget_io_init.h"

#define SAR_ADC_INDEX       (0U)
#define SAR_ADC_CHANNEL     (0U)
#define SAR_ADC_SEQENCER    (0U)
#define SAR_ADC_VREF_MV     (1800U)

void adc_init(void)
{
    // Step 1: Initialize the AutAnalog block
    Cy_AutAnalog_Init(&autonomous_analog_init);  // BSP-generated config

    // Step 2: Set interrupt mask
    Cy_AutAnalog_SetInterruptMask(CY_AUTANALOG_INT_SAR0_RESULT);

    // Step 3: Start autonomous conversions
    Cy_AutAnalog_StartAutonomousControl();
}

int16_t adc_read_mv(void)
{
    int32_t counts = Cy_AutAnalog_SAR_ReadResult(SAR_ADC_INDEX,
        CY_AUTANALOG_SAR_INPUT_GPIO, SAR_ADC_CHANNEL);
    return Cy_AutAnalog_SAR_CountsTo_mVolts(SAR_ADC_INDEX, false,
        SAR_ADC_SEQENCER, CY_AUTANALOG_SAR_INPUT_GPIO,
        SAR_ADC_CHANNEL, SAR_ADC_VREF_MV, counts);
}
```

**PSOC Edge — HAL path (validated — bike-computer project):**
```c
#include "mtb_hal.h"
#include "cybsp.h"
#include "cy_autanalog.h"

static mtb_hal_adc_t adc_obj;
static mtb_hal_adc_channel_t adc_channel;
static mtb_hal_adc_channel_t *adc_channels[1];

void adc_init(void)
{
    Cy_AutAnalog_Init(&autonomous_analog_init);
    adc_channels[0] = &adc_channel;
    mtb_hal_adc_setup(&adc_obj, &CYBSP_SAR_ADC_hal_config, NULL, adc_channels);
    mtb_hal_adc_enable(&adc_obj, true);
    Cy_AutAnalog_StartAutonomousControl();
}

uint16_t adc_read(void)
{
    return mtb_hal_adc_read_u16(&adc_channel);  // 0-65535
}
```

**Two paths — when to use which:**

| Approach | When to use | Read returns |
|---|---|---|
| **PDL** (`Cy_AutAnalog_*`) | Simple ADC reads, official CE pattern, no middleware | Raw counts → explicit mV conversion |
| **HAL** (`mtb_hal_adc_*`) | Middleware needs `mtb_hal_adc_t*`, filtered reads, multi-channel apps | 0-65535 (uint16_t), or filtered values |

**Key differences from PSOC 6:**

- Both paths require `Cy_AutAnalog_Init()` + `Cy_AutAnalog_StartAutonomousControl()` — no equivalent on PSOC 6
- PDL path: `Cy_AutAnalog_SAR_ReadResult()` returns raw counts, convert with `Cy_AutAnalog_SAR_CountsTo_mVolts()`
- HAL path: `mtb_hal_adc_read_u16()` returns 0-65535 (not millivolts) — caller scales as needed
- HAL path has additional read functions: `read_latest()`, `read_multiple()`, `read_filtered()` with filter types (median, average, LPF, CIC, LIF)
- No `cyhal_adc_free()` / `cyhal_adc_channel_free()` equivalent — lifecycle managed by BSP

## §8 — PWM (Detailed Pattern)

**PSOC 6:**
```c
cyhal_pwm_t pwm_obj;
cyhal_pwm_init(&pwm_obj, P6_0, NULL);
cyhal_pwm_set_duty_cycle(&pwm_obj, 50.0f, 1000);  // 50% duty, 1kHz
cyhal_pwm_start(&pwm_obj);
```

**PSOC Edge — PDL path (validated — official Infineon pwm-square-wave CE):**
```c
#include "retarget_io_init.h"

void pwm_init(void)
{
    // TCPWM configured in Device Configurator (frequency, duty, pin all set there)
    Cy_TCPWM_PWM_Init(CYBSP_PWM_LED_CTRL_HW,
        CYBSP_PWM_LED_CTRL_NUM, &CYBSP_PWM_LED_CTRL_config);
    Cy_TCPWM_PWM_Enable(CYBSP_PWM_LED_CTRL_HW,
        CYBSP_PWM_LED_CTRL_NUM);
    Cy_TCPWM_TriggerStart_Single(CYBSP_PWM_LED_CTRL_HW,
        CYBSP_PWM_LED_CTRL_NUM);
}
```

**PSOC Edge — HAL path (for runtime frequency/duty changes):**
```c
#include "mtb_hal.h"
#include "cybsp.h"

static mtb_hal_pwm_t pwm_obj;

void pwm_init(void)
{
    // TCPWM configured in Device Configurator as PWM mode
    mtb_hal_pwm_setup(&pwm_obj, &MY_PWM_hal_config, NULL);
    mtb_hal_pwm_set_period(&pwm_obj, 1000, 500);  // 1000µs period, 500µs pulse = 50% duty at 1kHz
    mtb_hal_pwm_start(&pwm_obj);
}
```

**Two paths — when to use which:**

| Approach | When to use |
|---|---|
| **PDL** (`Cy_TCPWM_PWM_*`) | Fixed frequency/duty set in Device Configurator (LED blink, buzzer), official CE pattern |
| **HAL** (`mtb_hal_pwm_*`) | Runtime duty/frequency changes needed (motor control, dimming, audio), middleware integration |

**Key differences from PSOC 6:**

- PSOC 6 `cyhal_pwm_init(obj, pin, clk)` → PDL: Device Configurator + `Cy_TCPWM_PWM_Init(HW, NUM, config)`, or HAL: `mtb_hal_pwm_setup(obj, cfg, clk)`
- PSOC 6 `cyhal_pwm_set_duty_cycle(obj, duty_pct, freq_hz)` → HAL: `mtb_hal_pwm_set_period(obj, period_us, pulse_width_us)` — must convert
- PDL path has no runtime duty API — all config is in Device Configurator
- New HAL functions not in `cyhal`: `mtb_hal_pwm_pause()`, `mtb_hal_pwm_resume()`, `mtb_hal_pwm_reload()`
- Advanced: `set_period_count()`, `set_compare_count()`, `set_deadtime_count()` for raw counter control
- Complementary output: `mtb_hal_pwm_configure_output(obj, out, out_compl)` with 6 output source options
- No `cyhal_pwm_free()` equivalent
- Priority parameter removed from `enable_event` (was 4 params, now 3)

## §9 — Project Structure Changes

PSOC 6 usually uses a single application project. PSOC Edge E84 usually uses a multi-core structure:

```text
my-project/
├── proj_cm33_s/      # Secure core (usually untouched)
├── proj_cm33_ns/     # Non-secure CM33 (main application)
├── proj_cm55/        # CM55 (ML, display, or DSP tasks)
└── bsps/             # Shared BSP
```

Migration implications:

- Put most application code in `proj_cm33_ns/`.
- Keep secure boot / secure services in `proj_cm33_s/` unless you have a real secure-world requirement.
- Only move work into `proj_cm55/` when the application actually needs CM55 participation.
- Expect generated BSP and configurator files to be shared rather than living beside a single `main.c`.

## §10 — Makefile Differences

The important shift is not just `TARGET=`. It is also where build settings live.

### Common differences

| Area | Typical PSOC 6 project | Typical PSOC Edge E84 project |
|---|---|---|
| Target | `TARGET=...PSOC 6 kit...` | `TARGET=KIT_PSE84_EVAL_EPC2` or equivalent PSE84 BSP |
| Project shape | One Makefile drives one app | Root + per-core projects |
| Application build settings | Usually one place | Often `proj_cm33_ns/Makefile` for app logic |
| BLE component naming | Often `BLUETOOTH`-style assumptions in old examples | `WICED_BLE HCI-UART` |
| FreeRTOS define | Sometimes implicit | Add `DEFINES+=CY_RTOS_AWARE` when using FreeRTOS |
| retarget-io line endings | Often ignored | Add `DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF` |

### Practical migration snippet

```makefile
# Root or common target selection
TARGET=KIT_PSE84_EVAL_EPC2

# Application-side additions as needed
COMPONENTS+=FREERTOS
COMPONENTS+=WICED_BLE HCI-UART
COMPONENTS+=LWIP MBEDTLS SECURE_SOCKETS WCM

DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
DEFINES+=CY_RTOS_AWARE
DEFINES+=CYBSP_WIFI_CAPABLE

# BT UART baud rates (validated — bike-computer, wifi-ble projects)
DEFINES+=CYBSP_BT_PLATFORM_CFG_BAUD_RATE=CY_HAL_UART_BAUD_3000000
DEFINES+=CYBSP_BT_PLATFORM_CFG_BAUD_DOWNLOAD=3000000
DEFINES+=CYBSP_BT_PLATFORM_CFG_BAUD_FEATURE=3000000
```

Notes:

- Exact `COMPONENTS` depend on the middleware actually used.
- WiFi projects should use the current WiFi super-dependency strategy from the workspace reference docs, not ad hoc PSOC 6 dependency sets.
- If the project template has separate per-core Makefiles, update the application core Makefile, not just the root file.

## §11 — Library Compatibility

- Libraries using `cyhal_*.h` are **NOT compatible** with PSOC Edge without modification.
- `mtb-hal-cat1` does **NOT** contain PSE84 support. Dead code produces misleading errors.
- Known incompatible examples include `display-oled-ssd1306`, many community PSOC 6 examples, and any middleware that owns its own `cyhal_*_init(...)` sequence.
- `sensor-orientation-bmm350` needs version-by-version verification: PSOC 6-oriented HAL/I2C paths are migration candidates, while newer PDL/I3C-native paths may already be closer to PSOC Edge.

### Migration options

1. **Direct port** — replace types, APIs, and init model throughout the library
2. **Thin shim** — only for simple rename cases such as I2C write/read wrappers
3. **Alternative library** — use a PSOC Edge-native or PDL-based library instead

### Thin shim example (rename-only I2C case)

```c
#ifndef CYHAL_I2C_COMPAT_H
#define CYHAL_I2C_COMPAT_H

#include "mtb_hal_i2c.h"
typedef mtb_hal_i2c_t cyhal_i2c_t;

static inline cy_rslt_t cyhal_i2c_master_write(
    cyhal_i2c_t *obj,
    uint16_t addr,
    const uint8_t *data,
    uint16_t size,
    uint32_t timeout,
    bool send_stop)
{
    return mtb_hal_i2c_controller_write(obj, addr, data, size, timeout, send_stop);
}

#endif
```

Use shims only when the old library already accepts a pre-configured bus object. If the library calls `cyhal_i2c_init(...)` or `cyhal_uart_init(...)` internally, a shim is not enough.

## §12 — Common Build Errors After Migration

| Error | Likely Cause | Fix |
|---|---|---|
| `fatal error: cyhal.h: No such file or directory` | PSOC 6 HAL header left in source | Replace with `mtb_hal.h` or the matching `mtb_hal_*.h` header |
| `fatal error: cyhal_i2c.h: No such file or directory` | Old peripheral include still present | Replace include and port the API usage |
| `fatal error: triggers/cyhal_triggers_pse84.h: No such file or directory` | A library pulled in `mtb-hal-cat1` | Remove or port the offending `cyhal`-dependent library |
| `too many arguments to function 'mtb_hal_i2c_enable_event'` | Old PSOC 6 call kept the priority arg | Remove the priority parameter |
| `too many arguments to function 'mtb_hal_pwm_enable_event'` | Same pattern on PWM | Remove the priority parameter |
| `too many arguments to function 'mtb_hal_timer_enable_event'` | Same pattern on Timer | Remove the priority parameter |
| `error: 'mtb_hal_i2c_cfg_t' has no member named 'is_slave'` | Old config-field names | Use `is_target`, `frequency_hz`, and add new fields as needed |
| `undefined reference to cyhal_i2c_master_mem_write` | No PSE84 equivalent exists | Rewrite as manual register-address write plus data transfer |
| `error: 'CYBSP_*_hal_config' undeclared` | Device Configurator output not generated or peripheral not configured | Open/save the configurator design and include generated headers |
| `printf` builds but nothing prints | PSOC 6 retarget init model still in use | Replace with the 3-step retarget wrapper |

When debugging the last 10% of the port, search for remaining `cyhal_` tokens first. That usually identifies the next real blocker faster than reading the compiler output top to bottom.
