# Porting PSOC 6 Examples to PSOC Edge

Guide for migrating existing PSOC 6 (CY8C6xxx) code examples to PSOC Edge E84 (PSE84).

> Distilled from porting experiences across 5+ validated examples.

---

## Key Differences

| Aspect | PSOC 6 | PSOC Edge E84 |
|--------|--------|---------------|
| Project layout | Single-project (most) | 3-project (mandatory) |
| Application core | CM4 | CM33 NS |
| HAL API | `cyhal_` | `mtb_hal_` |
| HAL source | `mtb-hal-cat1` (standalone) | DSL repo (bundled) |
| TrustZone | Optional (PSOC 64 only) | Always present |
| Secondary core | CM0+ (usually sleeping) | CM55 (sleeping or active) |
| Printf | `cy_retarget_io_init()` works | Requires `--wrap=_write` |

---

## Step-by-Step Migration

### 1. Create 3-project structure

PSOC Edge requires `proj_cm33_s/`, `proj_cm33_ns/`, and `proj_cm55/` sub-projects.

**What was `main.c`** → moves to `proj_cm33_ns/main.c`
**What was `source/`** → moves to `proj_cm33_ns/source/`
**What was `include/`** → moves to `proj_cm33_ns/include/`
**What was `deps/`** → moves to `proj_cm33_ns/deps/`

`proj_cm33_s/` and `proj_cm55/` use boilerplate — copy from any PSOC Edge code example.

### 2. Update Makefile

**Root Makefile:**
```makefile
MTB_TYPE=APPLICATION
MTB_PROJECTS=proj_cm33_s proj_cm33_ns proj_cm55
```

**common.mk:**
```makefile
TARGET=KIT_PSE84_EVAL_EPC2    # Was CY8CKIT-062S2-43012 or similar
TOOLCHAIN=GCC_ARM
```

### 3. Migrate HAL API calls

| PSOC 6 (`cyhal_`) | PSOC Edge (`mtb_hal_`) |
|--------------------|-----------------------|
| `cyhal_gpio_init()` | `mtb_hal_gpio_init()` |
| `cyhal_uart_init()` | `mtb_hal_uart_init()` |
| `cyhal_i2c_init()` | `mtb_hal_i2c_init()` |
| `cyhal_spi_init()` | `mtb_hal_spi_init()` |
| `cyhal_pwm_init()` | `mtb_hal_pwm_init()` |
| `cyhal_adc_init()` | `mtb_hal_adc_init()` |
| `cyhal_timer_init()` | `mtb_hal_timer_init()` |

**Important:** The function signatures may differ slightly. Check the DSL repo API docs:
- `https://github.com/Infineon/mtb-dsl-pse8xxgo`
- `https://github.com/Infineon/mtb-dsl-pse8xxgp`

Not all `cyhal_` functions have `mtb_hal_` equivalents. Fall back to PDL (`Cy_`) when needed.

### 4. Fix printf

Add to `proj_cm33_ns/Makefile`:
```makefile
LDFLAGS+=-Wl,--wrap=_write
```
And add the `__wrap__write()` function. See `#retarget-io-fix` for the complete pattern.

### 5. Update library .mtb files

Most WiFi/BT/MQTT middleware libraries work on both PSOC 6 and PSOC Edge. However:
- Check each library's README for PSOC Edge compatibility
- Some library versions have PSOC Edge-specific branches
- The BSP `.mtb` file must change to the PSOC Edge BSP

### 6. Handle TrustZone

PSOC Edge always boots through TrustZone secure world. The `proj_cm33_s/` sub-project handles this automatically via BSP boilerplate. Your application code in `proj_cm33_ns/` runs in the non-secure world.

**What changes:**
- `cybsp_init()` is called in `proj_cm33_s/main.c`, NOT in your app
- Your app should NOT call `cybsp_init()` again in `proj_cm33_ns/main.c`
- Certain peripherals may need non-secure configuration (handled by BSP)

### 7. Update COMPONENTS and DEFINES

Most COMPONENTS names are the same. Check for PSOC Edge-specific requirements:
```makefile
# These typically work unchanged
COMPONENTS+=FREERTOS LWIP MBEDTLS

# WiFi capability define
DEFINES+=CYBSP_WIFI_CAPABLE
```

---

## Common Porting Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Single-project structure | Build errors on PSOC Edge | Convert to 3-project layout |
| `cyhal_` API calls | Undefined symbol errors | Replace with `mtb_hal_` equivalents |
| `cybsp_init()` in app | Double-init crash or TZ fault | Remove from NS main — BSP handles it |
| Printf not working | No serial output | Add `--wrap=_write` linker flag |
| Missing BSP | Target not recognized | Use correct PSOC Edge BSP `.mtb` |
| `cyhal_` timer/crypto | No `mtb_hal_` equivalent | Use PDL (`Cy_`) directly |

---

## Library Compatibility

Most Infineon middleware works on both PSOC 6 and PSOC Edge:

| Library | PSOC 6 | PSOC Edge | Notes |
|---------|--------|-----------|-------|
| FreeRTOS | ✓ | ✓ | Same library |
| WiFi Connection Manager | ✓ | ✓ | Same library |
| MQTT | ✓ | ✓ | Same library |
| lwIP | ✓ | ✓ | Same library |
| Mbed TLS | ✓ | ✓ | Same library |
| Secure Sockets | ✓ | ✓ | Same library |
| BTSTACK | ✓ | ✓ | Check version compatibility |
| retarget-io | ✓ | ✓ | Needs `--wrap=_write` on Edge |
| HTTP Client | ✓ | ✓ | Same library |
| Emulated EEPROM | ✓ | Check | May need Edge-specific version |

---

## Testing the Port

1. `make getlibs` — verify all libraries resolve
2. `make build` — check for compilation errors (HAL API changes)
3. `make program` — flash and verify serial output works
4. Test all peripherals — especially any that use HAL (pins, UART, I2C, SPI)
5. Test WiFi/BT connectivity — should work if libraries are compatible
