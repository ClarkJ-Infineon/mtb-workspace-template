---
description: "Generate a Device Configurator setup specification for a peripheral — tells the developer exactly what to configure in the MTB IDE"
---

Generate a Device Configurator configuration specification for the requested peripheral. Copilot cannot create `.cycfg` files directly — the developer must apply this specification in the ModusToolbox™ IDE Device Configurator GUI.

Read `CONTEXT.md` first to confirm:
- Target device family and specific device
- Kit name (for BSP pin aliases like `CYBSP_USER_LED`)

Then produce a configuration document in this format:

---

## Device Configurator Setup — [Peripheral Name]

**Target:** [device and kit from CONTEXT.md]
**File to open in MTB IDE:** `[project]/design.modus` (or the relevant sub-project for PSOC Edge: `proj_cm33_ns/design.modus`)

### [Peripheral Block Name] Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Peripheral | [e.g., SCB0 / UART0 / GPIO_PRT0] | [which block to use] |
| Mode | [e.g., UART / I2C Master / PWM] | |
| [Setting name] | [value] | [why this value] |

**Pin assignments:**

| Signal | Pin | BSP alias (if available) |
|--------|-----|--------------------------|
| [e.g., TX] | [e.g., P5.1] | `CYBSP_DEBUG_UART_TX` |
| [e.g., RX] | [e.g., P5.0] | `CYBSP_DEBUG_UART_RX` |

**After configuring in Device Configurator:**
- Save the design file — this regenerates `GeneratedSource/`
- Add the generated init call to `main.c`: `[e.g., cybsp_init()]`
- The BSP alias macros are defined in `cybsp.h`

---

**Device-family notes:**
- PSOC Edge: configure in `proj_cm33_ns/design.modus`; secure-world peripherals go in `proj_cm33_s/design.modus`
- XMC1000/4000: peripheral names use XMC convention (e.g., `USIC0 CH0` for UART); refer to the device datasheet for pin mux options
- XMC7000: same Device Configurator flow as PSOC 6
- All families: `GeneratedSource/` is auto-generated — do NOT manually edit files in that folder

If pin aliases (`CYBSP_*`) are not available for the target peripheral, specify the physical pin name (e.g., `P5.1`) and note that the developer should verify against the kit schematic.
