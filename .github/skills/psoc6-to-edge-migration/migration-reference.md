# PSOC 6 → PSOC Edge E84 Migration Reference

Companion reference for the `psoc6-to-edge-migration` skill.

Use this file when you need the deeper API map while porting ModusToolbox™ projects from PSOC 6 `cyhal_*` code to PSOC Edge E84 `mtb_hal_*` + PDL code.

> **Two migration paths:** Official Infineon PSOC Edge code examples use **pure PDL** for peripheral access. Custom/middleware-heavy projects use **HAL wrappers** (`mtb_hal_*`). Both are valid — this reference documents both. See the PDL vs HAL guidance table in SKILL.md §3 for selection criteria.

## 1 — I2C

### 1.1 Core migration rule

I2C is the clearest example of the PSOC 6 → PSOC Edge E84 architecture shift:

- PSOC 6 creates an I2C instance from SDA/SCL pins at runtime.
- PSOC Edge E84 expects Device Configurator-generated SCB objects, then explicit PDL init, then `mtb_hal_i2c_setup(...)`.
- Terminology changes from **master/slave** to **controller/target**.
- `mem_write` / `mem_read` helpers are removed.

### 1.2 Complete function mapping table

| PSOC 6 (`cyhal_i2c_*`) | PSOC Edge (`mtb_hal_i2c_*`) | Notes |
|---|---|---|
| `cyhal_i2c_init(obj, sda, scl, clk)` | `Cy_SCB_I2C_Init(...)` + `Cy_SCB_I2C_Enable(...)` + `mtb_hal_i2c_setup(obj, hal_cfg, ctx, clk)` | Completely different init model |
| `cyhal_i2c_configure(obj, cfg)` | `mtb_hal_i2c_configure(obj, cfg)` | Same role, renamed types/fields |
| `cyhal_i2c_master_write(obj, addr, data, size, timeout, stop)` | `mtb_hal_i2c_controller_write(obj, addr, data, size, timeout, stop)` | Rename only |
| `cyhal_i2c_master_read(obj, addr, data, size, timeout, stop)` | `mtb_hal_i2c_controller_read(obj, addr, data, size, timeout, stop)` | Rename only |
| `cyhal_i2c_master_mem_write(...)` | ❌ Removed | Manually prepend register address to TX buffer |
| `cyhal_i2c_master_mem_read(...)` | ❌ Removed | Write register address, then do separate read |
| `cyhal_i2c_master_transfer_async(...)` | ❌ Removed from public migration path | Rework at application level |
| `cyhal_i2c_slave_config_read_buffer(obj, data, size)` | `mtb_hal_i2c_target_config_read_buffer(obj, data, size)` | `slave` → `target` |
| `cyhal_i2c_slave_config_write_buffer(obj, data, size)` | `mtb_hal_i2c_target_config_write_buffer(obj, data, size)` | `slave` → `target` |
| `cyhal_i2c_register_callback(obj, cb, arg)` | `mtb_hal_i2c_register_callback(obj, cb, arg)` | Same concept |
| — | `mtb_hal_i2c_register_address_callback(obj, cb, arg)` | New explicit address callback |
| — | `mtb_hal_i2c_register_byte_received_callback(obj, cb, arg)` | New explicit per-byte callback |
| `cyhal_i2c_enable_event(obj, event, intr_priority, enable)` | `mtb_hal_i2c_enable_event(obj, event, enable)` | Priority parameter removed |
| — | `mtb_hal_i2c_enable_address_event(obj, event, enable)` | New address-event control |
| `cyhal_i2c_slave_readable(obj)` | `mtb_hal_i2c_target_readable(obj)` | Rename |
| `cyhal_i2c_slave_writable(obj)` | `mtb_hal_i2c_target_writable(obj)` | Rename |
| `cyhal_i2c_slave_read(obj, dst, size, timeout)` | `mtb_hal_i2c_target_read(obj, dst, size, timeout)` | Rename |
| `cyhal_i2c_slave_write(obj, src, size, timeout)` | `mtb_hal_i2c_target_write(obj, src, size, timeout)` | Rename |
| `cyhal_i2c_slave_abort_read(obj)` | `mtb_hal_i2c_target_abort_read(obj)` | Rename |
| `cyhal_i2c_clear(obj)` | `mtb_hal_i2c_clear(obj)` | Same job |
| `cyhal_i2c_process_interrupt(obj)` | `mtb_hal_i2c_process_interrupt(obj)` | Same job |
| `cyhal_i2c_free(obj)` | ❌ No equivalent | Lifecycle managed by BSP / Configurator |

### 1.3 Type mapping

| PSOC 6 Type | PSOC Edge Type |
|---|---|
| `cyhal_i2c_t` | `mtb_hal_i2c_t` |
| `cyhal_i2c_cfg_t` | `mtb_hal_i2c_cfg_t` |
| `cyhal_i2c_event_t` | `mtb_hal_i2c_event_t` |
| `cyhal_i2c_event_callback_t` | `mtb_hal_i2c_event_callback_t` |
| `cyhal_i2c_addr_event_t` | `mtb_hal_i2c_addr_event_t` |

### 1.4 Config struct comparison

**PSOC 6 (`cyhal_i2c_cfg_t`)**

```c
typedef struct {
    bool     is_slave;
    uint16_t address;
    uint32_t frequencyhal_hz;
} cyhal_i2c_cfg_t;
```

**PSOC Edge (`mtb_hal_i2c_cfg_t`)**

```c
typedef struct {
    bool     is_target;
    uint16_t address;
    uint32_t frequency_hz;
    uint8_t  address_mask;
    bool     enable_address_callback;
} mtb_hal_i2c_cfg_t;
```

**Field migration**

| Old field | New field | Porting note |
|---|---|---|
| `is_slave` | `is_target` | Same meaning, new terminology |
| `address` | `address` | Same |
| `frequencyhal_hz` | `frequency_hz` | Rename |
| — | `address_mask` | New target-mode field |
| — | `enable_address_callback` | New target-mode field |

### 1.5 Init pattern side-by-side

**PSOC 6 Pattern:**
```c
cyhal_i2c_t i2c_obj;
cyhal_i2c_cfg_t cfg = { .is_slave = false, .frequencyhal_hz = 400000 };
cyhal_i2c_init(&i2c_obj, CYBSP_I2C_SDA, CYBSP_I2C_SCL, NULL);
cyhal_i2c_configure(&i2c_obj, &cfg);
```

**PSOC Edge — HAL Pattern:**
```c
mtb_hal_i2c_t i2c_obj;
cy_stc_scb_i2c_context_t i2c_context;
Cy_SCB_I2C_Init(CYBSP_I2C_HW, &CYBSP_I2C_config, &i2c_context);
Cy_SCB_I2C_Enable(CYBSP_I2C_HW, &i2c_context);
mtb_hal_i2c_setup(&i2c_obj, &CYBSP_I2C_hal_config, &i2c_context, NULL);
```

**PSOC Edge — PDL Pattern (validated — official i2c-controller-ezi2c-target CE):**

The official Infineon CE uses pure PDL for I2C, with byte-wise transfers:

```c
/* Controller init */
cy_stc_scb_i2c_context_t i2c_master_context;
Cy_SCB_I2C_Init(mI2C_HW, &mI2C_config, &i2c_master_context);
Cy_SCB_I2C_Enable(mI2C_HW);

/* Byte-wise write */
Cy_SCB_I2C_MasterSendStart(mI2C_HW, I2C_SLAVE_ADDR, CY_SCB_I2C_WRITE_XFER,
    TRANSFER_TIMEOUT_MS, &i2c_master_context);
Cy_SCB_I2C_MasterWriteByte(mI2C_HW, data_byte, TRANSFER_TIMEOUT_MS,
    &i2c_master_context);
Cy_SCB_I2C_MasterSendStop(mI2C_HW, TRANSFER_TIMEOUT_MS,
    &i2c_master_context);

/* Byte-wise read */
Cy_SCB_I2C_MasterSendStart(mI2C_HW, I2C_SLAVE_ADDR, CY_SCB_I2C_READ_XFER,
    TRANSFER_TIMEOUT_MS, &i2c_master_context);
Cy_SCB_I2C_MasterReadByte(mI2C_HW, CY_SCB_I2C_NAK, &read_byte,
    TRANSFER_TIMEOUT_MS, &i2c_master_context);
Cy_SCB_I2C_MasterSendStop(mI2C_HW, TRANSFER_TIMEOUT_MS,
    &i2c_master_context);
```

**EzI2C Target (PDL pattern):**
```c
/* Target init with interrupt */
cy_stc_scb_i2c_context_t i2c_slave_context;
Cy_SCB_EZI2C_Init(sEZI2C_HW, &sEZI2C_config, &i2c_slave_context);
cy_stc_sysint_t ezi2c_intr_cfg = { .intrSrc = sEZI2C_IRQ, .intrPriority = 7u };
Cy_SysInt_Init(&ezi2c_intr_cfg, ezi2c_isr);
NVIC_EnableIRQ(ezi2c_intr_cfg.intrSrc);
Cy_SCB_EZI2C_Enable(sEZI2C_HW);
```

**Two paths — when to use which:**

| Approach | When to use |
|---|---|
| **PDL byte-wise** | Simple sensor reads, official CE pattern, maximum control |
| **HAL** (`mtb_hal_i2c_*`) | Middleware needs `mtb_hal_i2c_t*`, buffer-based transfers, callbacks |

### 1.6 Event enums worth knowing

`mtb_hal_i2c_event_t` includes these target-side and controller-side flags:

- `MTB_HAL_I2C_TARGET_READ_EVENT`
- `MTB_HAL_I2C_TARGET_WRITE_EVENT`
- `MTB_HAL_I2C_TARGET_RD_IN_FIFO_EVENT`
- `MTB_HAL_I2C_TARGET_RD_BUF_EMPTY_EVENT`
- `MTB_HAL_I2C_TARGET_RD_CMPLT_EVENT`
- `MTB_HAL_I2C_TARGET_WR_CMPLT_EVENT`
- `MTB_HAL_I2C_TARGET_ERR_EVENT`
- `MTB_HAL_I2C_TARGET_RESTART_EVENT`
- `MTB_HAL_I2C_TARGET_STOP_EVENT`
- `MTB_HAL_I2C_TARGET_ARB_LOST_EVENT`
- `MTB_HAL_I2C_TARGET_TIMEOUT0_EVENT`
- `MTB_HAL_I2C_TARGET_TIMEOUT1_EVENT`
- `MTB_HAL_I2C_TARGET_TIMEOUT2_EVENT`
- `MTB_HAL_I2C_CONTROLLER_ERR_EVENT`
- `MTB_HAL_I2C_CONTROLLER_ARB_LOST_EVENT`
- `MTB_HAL_I2C_CONTROLLER_TIMEOUT0_EVENT`
- `MTB_HAL_I2C_CONTROLLER_TIMEOUT1_EVENT`
- `MTB_HAL_I2C_CONTROLLER_TIMEOUT2_EVENT`

Address event enum values:

- `MTB_HAL_I2C_GENERAL_CALL_EVENT`
- `MTB_HAL_I2C_ADDR_MATCH_EVENT`

### 1.7 `mem_write` / `mem_read` replacement pattern

**Old PSOC 6 style**

```c
cyhal_i2c_master_mem_write(&i2c_obj, dev_addr, &reg_addr, 1, data, len, timeout_ms);
cyhal_i2c_master_mem_read(&i2c_obj, dev_addr, &reg_addr, 1, data, len, timeout_ms);
```

**PSOC Edge replacement**

```c
/* mem_write replacement */
uint8_t tx_buf[1 + DATA_LEN];
tx_buf[0] = reg_addr;
memcpy(&tx_buf[1], data, DATA_LEN);
mtb_hal_i2c_controller_write(&i2c_obj, dev_addr, tx_buf, sizeof(tx_buf), timeout_ms, true);

/* mem_read replacement */
mtb_hal_i2c_controller_write(&i2c_obj, dev_addr, &reg_addr, 1u, timeout_ms, false);
mtb_hal_i2c_controller_read(&i2c_obj, dev_addr, data, DATA_LEN, timeout_ms, true);
```

## 2 — UART

### 2.1 Core migration rule

UART follows the same pattern as I2C:

- PSOC 6 often initializes from pins.
- PSOC Edge E84 uses Configurator-generated UART objects, explicit SCB UART init, then `mtb_hal_uart_setup(...)`.
- Event enable APIs lose the interrupt-priority parameter.

### 2.2 Full function list from the DSL header

| Function | Signature summary | Notes |
|---|---|---|
| `mtb_hal_uart_setup` | `(mtb_hal_uart_t* obj, const mtb_hal_uart_configurator_t* config, cy_stc_scb_uart_context_t* context, const mtb_hal_clock_t* clock)` | Declared in `mtb_hal_uart_impl.h` |
| `mtb_hal_uart_enable` | `(mtb_hal_uart_t* obj, bool enable)` | Enable/disable UART |
| `mtb_hal_uart_set_baud` | `(mtb_hal_uart_t* obj, uint32_t baudrate, uint32_t* actualbaud)` | `actualbaud` may be `NULL` |
| `mtb_hal_uart_get` | `(mtb_hal_uart_t* obj, uint8_t* value, uint32_t timeout)` | Blocking get with timeout |
| `mtb_hal_uart_put` | `(mtb_hal_uart_t* obj, uint32_t value)` | Blocking put of one byte |
| `mtb_hal_uart_readable` | `(mtb_hal_uart_t* obj)` → `uint32_t` | Bytes available to read |
| `mtb_hal_uart_writable` | `(mtb_hal_uart_t* obj)` → `uint32_t` | Space available to write |
| `mtb_hal_uart_clear` | `(mtb_hal_uart_t* obj)` | Clear UART buffers |
| `mtb_hal_uart_enable_cts_flow_control` | `(mtb_hal_uart_t* obj, bool enable)` | CTS flow control |
| `mtb_hal_uart_write` | `(mtb_hal_uart_t* obj, void* tx, size_t* tx_length)` | Synchronous TX |
| `mtb_hal_uart_read` | `(mtb_hal_uart_t* obj, void* rx, size_t* rx_length)` | Synchronous RX |
| `mtb_hal_uart_is_tx_active` | `(mtb_hal_uart_t* obj)` → `bool` | TX active state |
| `mtb_hal_uart_is_rx_active` | `(mtb_hal_uart_t* obj)` → `bool` | RX active state |
| `mtb_hal_uart_register_callback` | `(mtb_hal_uart_t* obj, mtb_hal_uart_event_callback_t callback, void* callback_arg)` | Register event callback |
| `mtb_hal_uart_enable_event` | `(mtb_hal_uart_t* obj, mtb_hal_uart_event_t event, bool enable)` | No priority parameter |
| `mtb_hal_uart_config_async` | `(mtb_hal_uart_t* obj, mtb_async_transfer_context_t* context)` | Conditional on async-transfer component |
| `mtb_hal_uart_config_async_dma` | `(mtb_hal_uart_t* obj, mtb_hal_dma_t* dma_rx, mtb_hal_dma_t* dma_tx, mtb_async_transfer_context_t* context)` | Conditional on async + DMA |
| `mtb_hal_uart_read_async` | `(mtb_hal_uart_t* obj, void* rx, size_t length)` | Async RX |
| `mtb_hal_uart_write_async` | `(mtb_hal_uart_t* obj, void* tx, size_t length)` | Async TX |
| `mtb_hal_uart_read_abort` | `(mtb_hal_uart_t* obj)` | Abort async RX |
| `mtb_hal_uart_write_abort` | `(mtb_hal_uart_t* obj)` | Abort async TX |
| `mtb_hal_uart_process_interrupt` | `(mtb_hal_uart_t* obj)` | Call from ISR |
| `mtb_hal_uart_write_string` | `(mtb_hal_uart_t* obj, const char* tx)` | Convenience string write |

Additional helper APIs also exist in the header for async availability checks:

- `mtb_hal_uart_is_async_rx_available(...)`
- `mtb_hal_uart_is_async_tx_available(...)`

### 2.3 UART event enum

`mtb_hal_uart_event_t` includes:

- `MTB_HAL_UART_IRQ_NONE`
- `MTB_HAL_UART_IRQ_TX_TRANSMIT_IN_FIFO`
- `MTB_HAL_UART_IRQ_TX_DONE`
- `MTB_HAL_UART_IRQ_RX_DONE`
- `MTB_HAL_UART_IRQ_RX_FULL`
- `MTB_HAL_UART_IRQ_RX_ERROR`
- `MTB_HAL_UART_IRQ_TX_ERROR`
- `MTB_HAL_UART_IRQ_RX_NOT_EMPTY`
- `MTB_HAL_UART_IRQ_TX_EMPTY`
- `MTB_HAL_UART_IRQ_TX_FIFO`
- `MTB_HAL_UART_IRQ_RX_FIFO`

### 2.4 Init pattern side-by-side

**PSOC 6 style**

```c
cyhal_uart_t uart_obj;
cyhal_uart_init(&uart_obj, tx_pin, rx_pin, NC, NC, NULL, NULL);
cyhal_uart_set_baud(&uart_obj, 115200, NULL);
```

**PSOC Edge style**

```c
mtb_hal_uart_t uart_obj;
cy_stc_scb_uart_context_t uart_context;

Cy_SCB_UART_Init(CYBSP_DEBUG_UART_HW, &CYBSP_DEBUG_UART_config, &uart_context);
Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);
mtb_hal_uart_setup(&uart_obj, &CYBSP_DEBUG_UART_hal_config, &uart_context, NULL);
mtb_hal_uart_set_baud(&uart_obj, 115200u, NULL);
```

### 2.5 Key porting differences

| PSOC 6 | PSOC Edge E84 |
|---|---|
| `cyhal_uart_init(...)` owns setup | PDL init + `mtb_hal_uart_setup(...)` |
| `cyhal_uart_getc(...)` / `putc(...)` | `mtb_hal_uart_get(...)` / `put(...)` |
| `enable_event(..., priority, enable)` | `enable_event(..., enable)` |
| Easy pin-based creation | Configurator-generated hardware mapping |

## 3 — ADC

### 3.1 Core migration rule

ADC on PSOC Edge E84 is not a drop-in SAR HAL port. The migrated flow depends on the PSE84 AutAnalog subsystem.

You must account for:

- AutAnalog PDL init before HAL use
- array-of-channel-pointers setup model
- `read_u16()` scaling semantics
- lack of `free()` APIs

### 3.2 Full function list from the DSL header

| Function | Signature summary | Notes |
|---|---|---|
| `mtb_hal_adc_setup` | `(mtb_hal_adc_t* obj, const mtb_hal_adc_configurator_t* config, mtb_hal_clock_t* clk, mtb_hal_adc_channel_t** channels)` | Channels passed as pointer array |
| `mtb_hal_adc_read_u16` | `(const mtb_hal_adc_channel_t* obj)` → `uint16_t` | Returns 0-65535 |
| `mtb_hal_adc_is_conversion_complete` | `(const mtb_hal_adc_channel_t* obj)` → `bool` | Poll completion |
| `mtb_hal_adc_read_latest` | `(const mtb_hal_adc_channel_t* obj, int32_t* result)` | Latest result |
| `mtb_hal_adc_read_multiple` | `(mtb_hal_adc_channel_t** channels, uint32_t num_channels, int32_t* result)` | Multi-channel read |
| `mtb_hal_adc_start_convert` | `(mtb_hal_adc_t* obj)` | Start conversion |
| `mtb_hal_adc_enable` | `(mtb_hal_adc_t* obj, bool enable)` | Enable/disable ADC |
| `mtb_hal_adc_is_ready` | `(mtb_hal_adc_t* obj)` → `bool` | Ready-state check |
| `mtb_hal_adc_read_filtered` | `(const mtb_hal_adc_channel_t* obj, mtb_hal_adc_filter_t filter, int32_t* result)` | Filtered read |

### 3.3 Filter types enum

`mtb_hal_adc_filter_t` includes:

- `MTB_HAL_ADC_FILTER_MEDIAN`
- `MTB_HAL_ADC_FILTER_LIF`
- `MTB_HAL_ADC_FILTER_AVG`
- `MTB_HAL_ADC_FILTER_CIC`
- `MTB_HAL_ADC_FILTER_LPF`

### 3.4 AutAnalog dependency

Both migration paths require the AutAnalog subsystem. The init sequence differs:

**PDL path (official CE):**
1. Device Configurator generates the analog config
2. `Cy_AutAnalog_Init(&autonomous_analog_init)`
3. `Cy_AutAnalog_SetInterruptMask(CY_AUTANALOG_INT_SAR0_RESULT)`
4. `Cy_AutAnalog_StartAutonomousControl()`
5. Read with `Cy_AutAnalog_SAR_ReadResult()` + `Cy_AutAnalog_SAR_CountsTo_mVolts()`

**HAL path:**
1. Device Configurator generates the analog config
2. `Cy_AutAnalog_Init(&autonomous_analog_init)`
3. `mtb_hal_adc_setup(&adc_obj, &cfg, NULL, channels)`
4. `mtb_hal_adc_enable(&adc_obj, true)`
5. `Cy_AutAnalog_StartAutonomousControl()`
6. Read with `mtb_hal_adc_read_u16(&channel)` or `mtb_hal_adc_read_filtered()`

### 3.5 Validated patterns — two approaches

**PDL path (official Infineon adc-basic CE):**

```c
#include "cybsp.h"
#include "retarget_io_init.h"

#define SAR_ADC_INDEX       (0U)
#define SAR_ADC_CHANNEL     (0U)
#define SAR_ADC_SEQENCER    (0U)
#define SAR_ADC_VREF_MV     (1800U)

void adc_init(void)
{
    Cy_AutAnalog_Init(&autonomous_analog_init);
    Cy_AutAnalog_SetInterruptMask(CY_AUTANALOG_INT_SAR0_RESULT);
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

**HAL path (from bike-computer project):**

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
    return mtb_hal_adc_read_u16(&adc_channel);  // 0-65535 (scaled to 16-bit)
}
```

### 3.6 PSOC 6 → PSOC Edge comparison

| Topic | PSOC 6 | PSOC Edge E84 (PDL) | PSOC Edge E84 (HAL) |
|---|---|---|---|
| Init | `cyhal_adc_init(...)` | `Cy_AutAnalog_Init(...)` + `SetInterruptMask` + `StartAutonomousControl` | `Cy_AutAnalog_Init(...)` + `mtb_hal_adc_setup(...)` + `mtb_hal_adc_enable(...)` + `StartAutonomousControl` |
| Channel creation | `cyhal_adc_channel_init_diff(...)` | Implicit in Device Configurator config | Channel pointer array passed to setup |
| Read | `cyhal_adc_read(ch)` → int32_t counts | `Cy_AutAnalog_SAR_ReadResult()` → raw counts | `mtb_hal_adc_read_u16(ch)` → 0-65535 |
| Read (voltage) | `cyhal_adc_read_uv(ch)` → microvolts (int32_t) | `Cy_AutAnalog_SAR_CountsTo_mVolts()` → millivolts (int16_t) | Caller must scale 0-65535 to voltage manually |
| Cleanup | `cyhal_adc_free(...)` / `cyhal_adc_channel_free(...)` | No equivalent | No equivalent |

⚠️ **Return value warning:** PSOC 6 `cyhal_adc_read_uv()` returns **microvolts** (int32_t). PSOC Edge PDL returns **millivolts** (int16_t). PSOC Edge HAL returns **raw 0-65535** (uint16_t). These are three different scales.

## 4 — PWM

### 4.1 Core migration rule

PWM moves from pin-owned runtime init to Configurator-owned TCPWM setup.

The biggest user-visible API change is duty-cycle programming:

- **PSOC 6:** duty percent + frequency
- **PSOC Edge:** explicit period and pulse width in microseconds

### 4.2 Full function list from the DSL header

| Function | Signature summary | Notes |
|---|---|---|
| `mtb_hal_pwm_setup` | `(mtb_hal_pwm_t* obj, const mtb_hal_pwm_configurator_t* config, const mtb_hal_clock_t* clock)` | Setup from Configurator output |
| `mtb_hal_pwm_set_period` | `(mtb_hal_pwm_t* obj, uint32_t period_us, uint32_t pulse_width_us)` | Main migration API |
| `mtb_hal_pwm_enable` | `(mtb_hal_pwm_t* obj, bool enable)` | Enable/disable |
| `mtb_hal_pwm_start` | `(mtb_hal_pwm_t* obj)` | Start PWM |
| `mtb_hal_pwm_stop` | `(mtb_hal_pwm_t* obj)` | Stop PWM |
| `mtb_hal_pwm_set_period_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Raw count-domain control |
| `mtb_hal_pwm_set_period_alt_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Alternate period count |
| `mtb_hal_pwm_set_compare_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Compare count |
| `mtb_hal_pwm_set_compare_shadow_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Shadow compare count |
| `mtb_hal_pwm_set_compare_alt_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Alternate compare count |
| `mtb_hal_pwm_set_compare_alt_shadow_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Alternate shadow compare |
| `mtb_hal_pwm_set_deadtime_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Deadtime count |
| `mtb_hal_pwm_set_deadtime_shadow_count` | `(mtb_hal_pwm_t* obj, uint32_t count)` | Shadow deadtime |
| `mtb_hal_pwm_configure_output` | `(mtb_hal_pwm_t* obj, mtb_hal_pwm_output_t out, mtb_hal_pwm_output_t out_compl)` | Output routing |
| `mtb_hal_pwm_resume` | `(mtb_hal_pwm_t* obj)` | Resume counter |
| `mtb_hal_pwm_pause` | `(mtb_hal_pwm_t* obj)` | Pause counter |
| `mtb_hal_pwm_reload` | `(mtb_hal_pwm_t* obj)` | Reload counter |
| `mtb_hal_pwm_register_callback` | `(mtb_hal_pwm_t* obj, mtb_hal_pwm_event_callback_t callback, void* callback_arg)` | Register callback |
| `mtb_hal_pwm_enable_event` | `(mtb_hal_pwm_t* obj, mtb_hal_pwm_event_t event, bool enable)` | No priority arg |
| `mtb_hal_pwm_process_interrupt` | `(mtb_hal_pwm_t* obj)` | ISR helper |

### 4.3 PWM event types

| Enum value | Meaning |
|---|---|
| `MTB_HAL_PWM_EVENT_NONE` | No event |
| `MTB_HAL_PWM_EVENT_TERMINAL_COUNT` | Terminal-count event |
| `MTB_HAL_PWM_EVENT_COMPARE` | Compare-match event |
| `MTB_HAL_PWM_EVENT_ALL` | Any PWM event |

### 4.4 PWM output source enum

| Enum value | Meaning |
|---|---|
| `MTB_HAL_PWM_OUTPUT_CONSTANT_0` | Force output low |
| `MTB_HAL_PWM_OUTPUT_CONSTANT_1` | Force output high |
| `MTB_HAL_PWM_OUTPUT_PWM_SIGNAL` | Normal PWM signal |
| `MTB_HAL_PWM_OUTPUT_INVERTED_PWM_SIGNAL` | Inverted PWM signal |
| `MTB_HAL_PWM_OUTPUT_PORT_DEFAULT` | Use port default behavior |
| `MTB_HAL_PWM_OUTPUT_SOURCE_MOTIF` | Source from MOTIF signal-conditioning path |

### 4.5 Validated patterns — two approaches

**PDL path (official Infineon pwm-square-wave CE):**

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

The PDL path puts ALL configuration (period, duty, alignment, pin assignment) in Device Configurator — there are zero runtime parameter calls.

**HAL path (for runtime frequency/duty changes):**

```c
#include "mtb_hal.h"
#include "cybsp.h"

static mtb_hal_pwm_t pwm_obj;

void pwm_init(void)
{
    mtb_hal_pwm_setup(&pwm_obj, &MY_PWM_hal_config, NULL);
    mtb_hal_pwm_set_period(&pwm_obj, 1000, 500);  // 1000µs period, 500µs pulse = 50% duty at 1kHz
    mtb_hal_pwm_start(&pwm_obj);
}
```

### 4.6 Duty-cycle conversion helper

```c
uint32_t period_us = 1000000u / frequency_hz;
uint32_t pulse_width_us = (uint32_t)(period_us * duty_percent / 100.0);
mtb_hal_pwm_set_period(&pwm_obj, period_us, pulse_width_us);
```

### 4.7 Breaking changes summary

| PSOC 6 | PSOC Edge E84 (PDL) | PSOC Edge E84 (HAL) |
|---|---|---|
| `cyhal_pwm_init(obj, pin, clk)` | DC + `Cy_TCPWM_PWM_Init(HW, NUM, cfg)` + `Enable` + `TriggerStart` | DC + `mtb_hal_pwm_setup(...)` |
| `cyhal_pwm_set_duty_cycle(obj, duty_pct, freq_hz)` | All in Device Configurator — no runtime API | `mtb_hal_pwm_set_period(obj, period_us, pulse_width_us)` |
| `enable_event(..., priority, enable)` | N/A (use PDL interrupt) | `enable_event(..., enable)` (priority removed) |
| Basic start/stop only | `Enable` + `TriggerStart` / `Disable` | Additional `pause`, `resume`, `reload`, raw count APIs |

## 5 — Timer

### 5.1 Core migration rule

Timer migration is mostly mechanical after setup, but runtime configuration is reduced.

The major behavioral change is this:

- `cyhal_timer_configure(...)` is part of the PSOC 6 runtime model.
- On PSOC Edge E84, timer shape is expected to come from Device Configurator.

### 5.2 Full function list from the DSL header

| Function | Signature summary | Notes |
|---|---|---|
| `mtb_hal_timer_setup` | `(mtb_hal_timer_t* obj, const mtb_hal_timer_configurator_t* config, mtb_hal_clock_t* clock)` | Setup from Configurator |
| `mtb_hal_timer_enable` | `(mtb_hal_timer_t* obj, bool enable)` | Enable/disable |
| `mtb_hal_timer_start` | `(mtb_hal_timer_t* obj)` | Start timer |
| `mtb_hal_timer_stop` | `(mtb_hal_timer_t* obj)` | Stop timer |
| `mtb_hal_timer_reset` | `(mtb_hal_timer_t* obj, uint32_t start_value)` | `start_value` is new |
| `mtb_hal_timer_read` | `(const mtb_hal_timer_t* obj)` → `uint32_t` | Read current value |
| `mtb_hal_timer_register_callback` | `(mtb_hal_timer_t* obj, mtb_hal_timer_event_callback_t callback, void* callback_arg)` | Register callback |
| `mtb_hal_timer_enable_event` | `(mtb_hal_timer_t* obj, mtb_hal_timer_event_t event, bool enable)` | No priority arg |
| `mtb_hal_timer_process_interrupt` | `(mtb_hal_timer_t* obj)` | ISR helper |

### 5.3 Timer event types

| Enum value | Meaning |
|---|---|
| `MTB_HAL_TIMER_EVENT_NONE` | No event |
| `MTB_HAL_TIMER_EVENT_TERMINAL_COUNT` | Counter reached terminal count |
| `MTB_HAL_TIMER_EVENT_COMPARE_CC0` | Compare 0 event |
| `MTB_HAL_TIMER_EVENT_COMPARE_CC0_OR_TERMINAL_COUNT` | Combined compare 0 / terminal event |
| `MTB_HAL_TIMER_EVENT_COMPARE_CC1` | Compare 1 event |
| `MTB_HAL_TIMER_EVENT_ALL` | All timer events |

### 5.4 Side-by-side comparison

| PSOC 6 | PSOC Edge E84 |
|---|---|
| `cyhal_timer_init(obj, clk)` | `mtb_hal_timer_setup(obj, cfg, clk)` |
| `cyhal_timer_start(obj)` | `mtb_hal_timer_start(obj)` |
| `cyhal_timer_stop(obj)` | `mtb_hal_timer_stop(obj)` |
| `cyhal_timer_read(obj)` | `mtb_hal_timer_read(obj)` |
| `cyhal_timer_reset(obj)` | `mtb_hal_timer_reset(obj, start_value)` |
| `cyhal_timer_configure(obj, cfg)` | No direct runtime equivalent |

### 5.5 Reset example

```c
/* Reset timer back to zero start value */
mtb_hal_timer_reset(&timer_obj, 0u);
```

### 5.6 Basic PSOC Edge timer pattern

```c
#include "mtb_hal.h"
#include "cybsp.h"

static mtb_hal_timer_t timer_obj;

void timer_init(void)
{
    mtb_hal_timer_setup(&timer_obj, &MY_TIMER_hal_config, NULL);
    mtb_hal_timer_start(&timer_obj);
}
```

## 6 — GPIO

### 6.1 Why this guide uses PDL for simple GPIO

For simple migrated PSOC 6 application code on PSOC Edge E84, GPIO is simplest and most reliable through PDL because:

- Device Configurator already owns pin mux and pad configuration
- BSP macros already expose port/pin pairs
- there is no direct one-call replacement for `cyhal_gpio_init(pin, ...)` semantics that preserves the old runtime-pin model

### 6.2 PDL pattern — simple I/O (most migration cases)

```c
#include "cy_pdl.h"
#include "cybsp.h"

/* Output */
Cy_GPIO_Write(CYBSP_USER_LED1_PORT, CYBSP_USER_LED1_PIN, 0u);

/* Input */
uint32_t pressed = Cy_GPIO_Read(CYBSP_USER_BTN1_PORT, CYBSP_USER_BTN1_PIN);

/* Toggle */
Cy_GPIO_Inv(CYBSP_USER_LED2_PORT, CYBSP_USER_LED2_PIN);
```

### 6.3 HSIOM override — pin mux change at runtime

Some pins may need their HSIOM function changed at runtime (e.g., switching from peripheral to GPIO mode for direct LED control in BLE examples):

```c
// Validated — ble-led project, led_task.c
Cy_GPIO_SetHSIOM(CYBSP_USER_LED3_PORT, CYBSP_USER_LED3_PIN, HSIOM_SEL_GPIO);
Cy_GPIO_Write(CYBSP_USER_LED3_PORT, CYBSP_USER_LED3_PIN, value);
```

This has no PSOC 6 `cyhal` equivalent — it is a PDL-only operation that may be needed when the Device Configurator has a pin assigned to a peripheral but the application needs to override it.

### 6.4 HAL GPIO objects — middleware integration

WiFi, BLE, and some sensor middleware require `mtb_hal_gpio_t` objects rather than raw PORT/PIN values. Use `mtb_hal_gpio_setup()` to create HAL objects from BSP-defined port/pin numbers:

```c
// Validated — wifi-ble project, wifi_manager.c
mtb_hal_gpio_setup(&wcm_config.wifi_wl_pin,
    CYBSP_WIFI_WL_REG_ON_PORT_NUM, CYBSP_WIFI_WL_REG_ON_PIN);
mtb_hal_gpio_setup(&wcm_config.wifi_host_wake_pin,
    CYBSP_WIFI_HOST_WAKE_PORT_NUM, CYBSP_WIFI_HOST_WAKE_PIN);

// Then use HAL write for the middleware-owned pin
mtb_hal_gpio_write(&wcm_config.wifi_wl_pin, 0);  // Assert reset
Cy_SysLib_Delay(2);
mtb_hal_gpio_write(&wcm_config.wifi_wl_pin, 1);  // Release reset
```

### 6.5 Choosing PDL vs HAL for GPIO

| Scenario | Use | Example |
|---|---|---|
| LED on/off, button read | PDL: `Cy_GPIO_Write/Read` | All simple application I/O |
| Toggle output | PDL: `Cy_GPIO_Inv` | Heartbeat LED |
| Pin mux override at runtime | PDL: `Cy_GPIO_SetHSIOM` | BLE LED reconfiguration |
| WiFi/BLE middleware pins | HAL: `mtb_hal_gpio_setup` + `mtb_hal_gpio_write` | WCM config, host wake |
| Sensor middleware bus reset | HAL: `mtb_hal_gpio_setup` + `mtb_hal_gpio_write` | Sensor power cycling |

### 6.6 Common mapping

| PSOC 6 | PSOC Edge E84 |
|---|---|
| `cyhal_gpio_init(pin, dir, drive, val)` | No equivalent in this migration guide; configure in Device Configurator |
| `cyhal_gpio_write(pin, val)` | `Cy_GPIO_Write(PORT, PIN, val)` or `mtb_hal_gpio_write(obj, val)` |
| `cyhal_gpio_read(pin)` | `Cy_GPIO_Read(PORT, PIN)` |
| `cyhal_gpio_toggle(pin)` | `Cy_GPIO_Inv(PORT, PIN)` |
| N/A | `Cy_GPIO_SetHSIOM(PORT, PIN, func)` — runtime pin mux |
| N/A | `mtb_hal_gpio_setup(obj, port_num, pin)` — create HAL object |

### 6.7 GPIO interrupt migration

PSOC 6 uses the `cyhal_gpio_register_callback()` + `cyhal_gpio_enable_event()` model. On PSOC Edge, wire interrupts through PDL + NVIC directly. See §6a below for the full pattern.

## 6a — Interrupt Setup Migration

### 6a.1 Core migration rule

PSOC 6 buries interrupt configuration inside `cyhal_*_register_callback()` + `cyhal_*_enable_event()`. On PSOC Edge, interrupt wiring is more explicit — you use PDL `Cy_SysInt_Init()` + NVIC directly, and ISR handlers typically forward to HAL `process_interrupt` helpers.

### 6a.2 PSOC 6 pattern (hidden behind HAL)

```c
// PSOC 6 — interrupts managed by cyhal
cyhal_gpio_register_callback(CYBSP_USER_BTN, &btn_callback_data);
cyhal_gpio_enable_event(CYBSP_USER_BTN, CYHAL_GPIO_IRQ_FALL, 3, true);
```

### 6a.3 PSOC Edge pattern (explicit PDL + NVIC)

```c
// Validated — wifi-ble project, wifi_manager.c
#include "cy_sysint.h"

// Step 1: Define ISR config struct
static const cy_stc_sysint_t sdio_intr_cfg = {
    .intrSrc = CYBSP_WIFI_SDIO_IRQ,       // BSP-generated IRQ number
    .intrPriority = APP_SDIO_INTERRUPT_PRIORITY
};

// Step 2: Write ISR handler — forward to HAL process_interrupt
static void sdio_interrupt_handler(void)
{
    mtb_hal_sdio_process_interrupt(&sdio_instance);
}

// Step 3: Register and enable
Cy_SysInt_Init(&sdio_intr_cfg, sdio_interrupt_handler);
NVIC_EnableIRQ(CYBSP_WIFI_SDIO_IRQ);
```

### 6a.4 ISR forwarding pattern

Most PSOC Edge ISR handlers are thin wrappers that call the appropriate HAL `process_interrupt` function. This pattern applies to all peripherals with interrupt support:

```c
// Generic pattern for any peripheral with HAL interrupt support
static void my_peripheral_isr(void)
{
    mtb_hal_<peripheral>_process_interrupt(&my_obj);
}
```

Available `process_interrupt` functions: `mtb_hal_i2c_process_interrupt`, `mtb_hal_spi_process_interrupt`, `mtb_hal_uart_process_interrupt`, `mtb_hal_pwm_process_interrupt`, `mtb_hal_timer_process_interrupt`, `mtb_hal_sdio_process_interrupt`.

### 6a.5 Migration mapping

| PSOC 6 | PSOC Edge E84 |
|---|---|
| `cyhal_*_register_callback(obj, cb_data)` | `Cy_SysInt_Init(&cfg, handler)` + HAL callback registration |
| `cyhal_*_enable_event(obj, event, priority, enable)` | `NVIC_EnableIRQ(irq)` — priority set in `cy_stc_sysint_t` struct |
| Priority as function parameter | Priority in `cy_stc_sysint_t.intrPriority` |
| IRQ number hidden from user | IRQ number explicit via BSP macro (e.g., `CYBSP_WIFI_SDIO_IRQ`) |

### 6a.6 Key differences

- **IRQ numbers are explicit** — use BSP-generated `CYBSP_*_IRQ` macros
- **Priority is structural** — set in the `cy_stc_sysint_t` struct, not passed as a function parameter
- **ISR registration is separate from event enabling** — unlike PSOC 6 where one call does both
- **HAL callbacks still work** for some peripherals via `mtb_hal_*_register_callback()` + `mtb_hal_*_enable_event()`, but the NVIC wiring must happen separately

## 6b — I2C Middleware Integration Pattern

### 6b.1 When you need this

Many PSOC 6 projects use middleware (sensor libraries, display drivers) that sit on top of I2C. On PSOC Edge, the middleware expects a pre-configured `mtb_hal_i2c_t` object rather than calling `cyhal_i2c_init()` internally.

### 6b.2 Validated pattern — sensor-hub project (BMI270 IMU)

```c
// Validated — sensor-hub project, imu_task.c
#include "mtb_hal.h"
#include "cybsp.h"
#include "mtb_bmi270.h"

static mtb_hal_i2c_t i2c_obj;
static cy_stc_scb_i2c_context_t i2c_context;
static mtb_bmi270_t bmi270_dev;

void imu_init(void)
{
    // Step 1: HAL setup with Device Configurator config
    cy_rslt_t result = mtb_hal_i2c_setup(&i2c_obj,
        &CYBSP_I2C_CONTROLLER_hal_config, &i2c_context, NULL);

    // Step 2: Configure bus parameters
    mtb_hal_i2c_cfg_t i2c_cfg = {
        .is_target = false,
        .address = 0,
        .frequency_hz = 400000U,
    };
    result = mtb_hal_i2c_configure(&i2c_obj, &i2c_cfg);

    // Step 3: Pass pre-configured HAL object to middleware
    result = mtb_bmi270_init_i2c(&bmi270_dev, &i2c_obj,
        MTB_BMI270_ADDRESS_DEFAULT);
}
```

### 6b.3 Migration pattern for PSOC 6 middleware users

| PSOC 6 pattern | PSOC Edge pattern |
|---|---|
| Middleware calls `cyhal_i2c_init()` internally | Middleware accepts pre-configured `mtb_hal_i2c_t*` |
| One-step: middleware owns bus setup + device init | Two-step: app owns bus setup, passes to middleware |
| Pin selection in middleware or app code | Pin selection in Device Configurator |

## 7 — BSP Dependency Chain

### 7.1 The chain that matters

```text
TARGET_KIT_PSE84_EVAL_EPC2 (BSP)
  └── BSP_DEVICE_SUPPORT_LIB = mtb-dsl-pse8xxgp
        └── hal/ directory -> mtb_hal_*.h headers + source
  ❌ Does NOT reference mtb-hal-cat1
```

### 7.2 Implication

A normal PSOC Edge E84 project should get its HAL from the DSL package only.

If `mtb-hal-cat1` shows up in a PSE84 build, it was almost certainly pulled in by a library dependency that still assumes PSOC 6 `cyhal_*`.

### 7.3 The misleading error source

`mtb-hal-cat1` still contains dead PSE84 conditional code that references non-existent headers and pin-package files. That dead code is why developers see cryptic include failures instead of a clean message that PSOC Edge E84 uses `mtb_hal_*`.

## 8 — Known Incompatible Libraries

| Library / category | Why it breaks | Workaround |
|---|---|---|
| `display-oled-ssd1306` | Hard-coded `cyhal_i2c_t` and `cyhal_i2c_master_write(...)` usage | Thin I2C shim for rename-only calls + rework caller-side init to Device Configurator + `mtb_hal_i2c_setup(...)` |
| Community PSOC 6 examples | Pervasive `cyhal_*` usage and single-project assumptions | Port app code peripheral by peripheral using this guide |
| Libraries that call `cyhal_i2c_init(...)` internally | PSOC Edge E84 does not support runtime pin-based I2C init | Change library API to accept preconfigured `mtb_hal_i2c_t*` |
| Libraries that call `cyhal_uart_init(...)` internally | Same init-model mismatch | Change library API to accept preconfigured `mtb_hal_uart_t*` |
| Libraries that call `cyhal_pwm_init(...)` internally | PWM setup is Device Configurator-owned on PSOC Edge | Rework to accept preconfigured `mtb_hal_pwm_t*` or direct config objects |
| Libraries built around `cyhal_gpio_init(...)` | GPIO config no longer belongs in library runtime init | Push GPIO config into BSP / Device Configurator and switch runtime operations to PDL |
| `sensor-orientation-bmm350` | Depends on exact library revision and transport path | Audit the actual version; migrate HAL/I2C paths, but keep newer PDL/I3C-native paths if already PSE84-friendly |

### 8.1 Thin shim example

For rename-only I2C libraries, this pattern can be enough:

```c
#ifndef CYHAL_I2C_COMPAT_H
#define CYHAL_I2C_COMPAT_H

#include "mtb_hal_i2c.h"
typedef mtb_hal_i2c_t cyhal_i2c_t;

static inline cy_rslt_t cyhal_i2c_master_write(
    cyhal_i2c_t *obj,
    uint16_t dev_addr,
    const uint8_t *data,
    uint16_t size,
    uint32_t timeout,
    bool send_stop)
{
    return mtb_hal_i2c_controller_write(obj, dev_addr, data, size, timeout, send_stop);
}

static inline cy_rslt_t cyhal_i2c_master_read(
    cyhal_i2c_t *obj,
    uint16_t dev_addr,
    uint8_t *data,
    uint16_t size,
    uint32_t timeout,
    bool send_stop)
{
    return mtb_hal_i2c_controller_read(obj, dev_addr, data, size, timeout, send_stop);
}

#endif
```

That does **not** solve runtime init differences. It only helps when the library already accepts a prebuilt bus object.

## 9 — Boot Sequence and Init Ordering

### 9.1 Why this matters for migration

PSOC 6 projects typically have a simple init flow: `cybsp_init()` → `cy_retarget_io_init()` → `__enable_irq()` → application code. The order is flexible.

PSOC Edge E84 has stricter ordering requirements due to multi-core boot, BLE stack early init, and TrustZone. Getting this wrong causes race conditions that manifest as hard faults or silent failures.

### 9.2 Standard PSOC Edge boot sequence

```c
// Validated across: wifi-ble, bike-computer, sensor-hub
int main(void)
{
    // 1. BSP init — clocks, pins, power domains
    cy_rslt_t result = cybsp_init();
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    // 2. Retarget-IO — UART for printf (3-step wrapper)
    init_retarget_io();

    // 3. BLE early init (if BLE is used) — BEFORE __enable_irq()
    // ble_stack_early_init();

    // 4. Enable CM55 (if dual-core) — BEFORE __enable_irq() on some flows
    // Cy_SysEnableCM55(MXCM55, CY_CM55_APP_BOOT_ADDR, BOOT_WAIT_US);

    // 5. Enable interrupts — AFTER subsystem early init
    __enable_irq();

    // 6. Create FreeRTOS tasks
    // xTaskCreate(...)

    // 7. Start scheduler (does not return)
    // vTaskStartScheduler();

    // 8. If CM33 has no FreeRTOS tasks, park in deep sleep
    // for (;;) { Cy_SysPm_CpuEnterDeepSleep(CY_SYSPM_WAIT_FOR_INTERRUPT); }
}
```

### 9.3 Key ordering differences from PSOC 6

| Aspect | PSOC 6 | PSOC Edge E84 |
|---|---|---|
| `__enable_irq()` timing | Usually called early or implicitly | Must be called AFTER BLE/CM55 init |
| CM55 boot | N/A (single core or M4+M0+) | Explicit `Cy_SysEnableCM55()` call required |
| BLE stack init | Part of AIROC stack init | `ble_stack_early_init()` must run before IRQ |
| Core parking | N/A | Unused core enters `Cy_SysPm_CpuEnterDeepSleep()` |
| FreeRTOS entry | `vTaskStartScheduler()` after basic init | Same, but subsystem init must precede it |

### 9.4 CM55 boot pattern

```c
// Validated — bike-computer, sensor-hub
#include "cy_syspm.h"

#define CY_CM55_APP_BOOT_ADDR    (0x24100000UL)
#define APP_CM55_BOOT_WAIT_US    (200UL)

Cy_SysEnableCM55(MXCM55, CY_CM55_APP_BOOT_ADDR, APP_CM55_BOOT_WAIT_US);
```

Note: The CM55 boot address is board/BSP-specific. Use the value from the BSP or linker configuration.

## 10 — Deferred Topics

### 10.1 SPI (detailed, deferred)

SPI migration is intentionally deferred here. The API family change follows the same pattern as I2C/UART, but display and sensor SPI ports usually need more guidance around chip-select handling, transfer helpers, and Device Configurator personalities.

### 10.2 Low-Power / syspm migration (deferred)

Low-power migration is not covered in detail in this skill. `cyhal_syspm_*` assumptions from PSOC 6 should not be treated as a direct porting surface.

### 10.3 DMA changes (deferred)

DMA-backed async transfer details are only lightly covered via the UART async APIs listed above. A full migration guide should cover DMA object ownership, configurator setup, and ISR routing.

### 10.4 NVM / RRAM differences (partially covered elsewhere)

Nonvolatile-memory migration is only partially covered here. PSOC Edge E84 memory ownership and address-map assumptions differ from PSOC 6, especially once secure/non-secure partitioning enters the design.

### 10.5 What this means in practice

Use this skill for the main first-wave migration blockers:

- HAL family change
- init model change
- retarget-io change
- GPIO / I2C / UART / ADC / PWM / Timer porting
- project structure and Makefile shifts

Handle SPI, low-power, DMA, and NVM follow-on work as separate focused migration tasks.
