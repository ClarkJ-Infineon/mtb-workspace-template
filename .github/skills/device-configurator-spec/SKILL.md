---
name: device-configurator-spec
description: Generate a Device Configurator setup specification for a peripheral in the ModusToolbox IDE. Use when configuring peripherals in design.modus, setting up I2C/SPI/UART/GPIO/Timer hardware, or documenting Device Configurator settings.
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

---

## Memory Configuration (PSOC Edge E84)

In addition to peripheral configuration, Device Configurator manages **Memory Configuration** — how System SRAM and SOCMEM are partitioned between cores. This is a separate personality from peripheral setup.

### When Memory Configuration Is Needed

Generate a memory configuration spec whenever the project uses capabilities that exceed BSP default allocations:

| Feature | Impact | Region to Adjust |
|---|---|---|
| WiFi + TLS on CM33 | ~530 KB data needed | Shrink `m33_code`, grow `m33_data` |
| LVGL display on CM55 | 2-3 MB for framebuffers + GPU | Grow `gfx_mem` in SOCMEM |
| Dual-core IPC | 4-32 KB shared memory | Verify `m33_m55_shared` exists and sized |
| ML inference on CM55 | Large model weights/activations | Grow `m55_data` or `m55_data_secondary` |

### Memory Configuration Spec Format

```markdown
## Device Configurator Setup — Memory Configuration

**File to open:** `design.modus` → Memory Configuration tab

### System SRAM (1 MB total = 0x100000)

| Region | Current Size | Recommended Size | Reason |
|--------|-------------|-----------------|--------|
| m33_code | [hex] ([KB] KB) | [hex] ([KB] KB) | [reason] |
| m33_data | [hex] ([KB] KB) | [hex] ([KB] KB) | [reason] |

### SOCMEM (5 MB total = 0x500000)

| Region | Current Size | Recommended Size | Reason |
|--------|-------------|-----------------|--------|
| gfx_mem | [hex] ([KB] KB) | [hex] ([KB] KB) | [reason] |

**Verification after changes:**
- System SRAM regions must total 0x100000 (1 MB)
- SOCMEM regions must total 0x500000 (5 MB)
- All regions contiguous (no address gaps)
- Clean rebuild required: `make clean && make build`
```

### Key Rules

- Regions within each memory block are **contiguous** — resizing one shifts neighbors
- Provide hex values alongside KB for easy Device Configurator entry
- Total must match hardware: System SRAM = `0x100000`, SOCMEM = `0x500000`
- Generated linker scripts in `GeneratedSource/` are updated on save — never edit manually
