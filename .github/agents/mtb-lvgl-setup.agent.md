---
name: mtb-lvgl-setup
description: Add LVGL v9 touchscreen graphics with VG-Lite GPU acceleration to a PSOC Edge E84 ModusToolbox project. Covers display drivers, touch input, Device Configurator GFXSS personality, lv_conf.h, framebuffer layout, and the full initialization sequence.
tools: ["read", "edit", "create", "search", "shell"]
---

# Add LVGL Graphics to PSOC Edge E84 Project

You are an expert in integrating LVGL v9 touchscreen graphics with VG-Lite GPU acceleration on PSOC Edge E84.

> **CRITICAL prerequisite:** The project MUST have been created with `project-creator-cli` from an LVGL-capable template. The Device Configurator GFXSS personality generates symbols (`GFXSS_DC_IRQ`, `GFXSS_GPU_IRQ`, `GFXSS_config`, I2C controller symbols) that cannot be replicated manually. See `mtb-project-creation` agent.

## Strategy: Start from the Harder Capability

The recommended approach is to start from the **Infineon LVGL demo CE** (`mtb-example-psoc-edge-lvgl-demo`) via `project-creator-cli`, then add your application logic on top. LVGL + GFXSS + VG-Lite requires BSP personalities, override files, and GPU memory sections that are difficult to retrofit. Other capabilities (WiFi, MQTT, BLE, sensors) are easier to add incrementally.

If adding LVGL to an existing project, copy the `bsps/` directory (specifically `design.modus` and generated source) from a working LVGL demo built for the same BSP.

---

## 1. Display Hardware Options

| Display | Size | Resolution | FB Res | Driver Library | Touch Library | `CONFIG_DISPLAY` |
|---|---|---|---|---|---|---|
| Waveshare 4.3" DSI | 4.3" | 800×480 | **832×480** ¹ | `display-dsi-waveshare-4-3-lcd` | `touch-ctp-ft5406` | `W4P3INCH_DISP` |
| Waveshare 7" DSI (C) | 7" | 1024×600 | 1024×600 | `display-dsi-waveshare-7-0-lcd-c` | `touch-ctp-gt911` | `WS7P0DSI_RPI_DISP` |
| EK79007AD3 10.1" TFT | 10.1" | 1024×600 | 1024×600 | `display-tft-ek79007ad3` | `touch-ctp-ili2511` | `WF101JTYAHMNB0_DISP` |

> ¹ 4.3" display: VG-Lite GPU requires stride alignment — 800px padded to 832px at RGB565. LVGL display created at 832×480, visible area clamped to 800×480 via `lv_display_set_resolution()`.

---

## 2. Dependencies — `proj_cm55/deps/` `.mtb` Files

### Core (always required)

```
# lvgl.mtb
https://github.com/lvgl/lvgl#v9.2.0#$$ASSET_REPO$$/lvgl/release-v9.2.0

# freertos.mtb
https://github.com/Infineon/freertos#release-v10.6.202#$$ASSET_REPO$$/freertos/release-v10.6.202

# retarget-io.mtb
https://github.com/cypresssemiconductorco/retarget-io#release-v1.9.0#$$ASSET_REPO$$/retarget-io/release-v1.9.0
```

### Display + touch drivers (include ALL three — unused ones are CY_IGNOREd)

```
# display-dsi-waveshare-4-3-lcd.mtb
https://github.com/Infineon/display-dsi-waveshare-4-3-lcd#release-v1.0.0#$$ASSET_REPO$$/display-dsi-waveshare-4-3-lcd/release-v1.0.0

# display-dsi-waveshare-7-0-lcd-c.mtb
https://github.com/Infineon/display-dsi-waveshare-7-0-lcd-c#release-v1.0.0#$$ASSET_REPO$$/display-dsi-waveshare-7-0-lcd-c/release-v1.0.0

# display-tft-ek79007ad3.mtb
https://github.com/Infineon/display-tft-ek79007ad3#release-v1.0.0#$$ASSET_REPO$$/display-tft-ek79007ad3/release-v1.0.0

# touch-ctp-ft5406.mtb
https://github.com/Infineon/touch-ctp-ft5406#release-v1.0.0#$$ASSET_REPO$$/touch-ctp-ft5406/release-v1.0.0

# touch-ctp-gt911.mtb
https://github.com/Infineon/touch-ctp-gt911#release-v1.0.0#$$ASSET_REPO$$/touch-ctp-gt911/release-v1.0.0

# touch-ctp-ili2511.mtb
https://github.com/Infineon/touch-ctp-ili2511#release-v1.0.0#$$ASSET_REPO$$/touch-ctp-ili2511/release-v1.0.0
```

Run `make getlibs` after creating all `.mtb` files.

---

## 3. Makefile Configuration

### `common.mk` (workspace root — shared by all cores)

```makefile
CONFIG_DISPLAY = W4P3INCH_DISP
COMPONENTS+=GFXSS
```

### `proj_cm55/Makefile`

```makefile
CORE=CM55
CORE_NAME=CM55_0

COMPONENTS+=FREERTOS RTOS_AWARE
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF _BAREMETAL=0 CY_RTOS_AWARE

# Display-conditional driver selection
ifeq ($(CONFIG_DISPLAY), WF101JTYAHMNB0_DISP)
DEFINES += MTB_DISPLAY_EK79007AD3 MTB_CTP_ILI2511
CY_IGNORE += $(SEARCH_display-dsi-waveshare-7-0-lcd-c) $(SEARCH_touch-ctp-gt911)
CY_IGNORE += $(SEARCH_display-dsi-waveshare-4-3-lcd) $(SEARCH_touch-ctp-ft5406)
else ifeq ($(CONFIG_DISPLAY), WS7P0DSI_RPI_DISP)
DEFINES += MTB_DISPLAY_WS7P0DSI_RPI MTB_CTP_GT911
CY_IGNORE += $(SEARCH_display-tft-ek79007ad3) $(SEARCH_touch-ctp-ili2511)
CY_IGNORE += $(SEARCH_display-dsi-waveshare-4-3-lcd) $(SEARCH_touch-ctp-ft5406)
else ifeq ($(CONFIG_DISPLAY), W4P3INCH_DISP)
DEFINES += MTB_DISPLAY_W4P3INCH_RPI MTB_CTP_FT5406
CY_IGNORE += $(SEARCH_display-tft-ek79007ad3) $(SEARCH_touch-ctp-ili2511)
CY_IGNORE += $(SEARCH_display-dsi-waveshare-7-0-lcd-c) $(SEARCH_touch-ctp-gt911)
endif

# LVGL: ignore unused modules and stock files replaced by Infineon overrides
CY_IGNORE += $(SEARCH_lvgl)/src/others/vg_lite_tvg $(SEARCH_lvgl)/src/libs/thorvg $(SEARCH_lvgl)/tests
CY_IGNORE += $(SEARCH_lvgl)/draw/nxp $(SEARCH_lvgl)/draw/renesas $(SEARCH_lvgl)/examples
CY_IGNORE += $(SEARCH_lvgl)/src/draw/vg_lite/lv_draw_vg_lite.c
CY_IGNORE += $(SEARCH_lvgl)/src/draw/vg_lite/lv_vg_lite_utils.c
CY_IGNORE += $(SEARCH_lvgl)/src/draw/vg_lite/lv_draw_vg_lite_img.c
CY_IGNORE += $(SEARCH_lvgl)/src/draw/sw/blend/helium $(SEARCH_lvgl)/src/draw/sw/blend/neon
CY_IGNORE += $(SEARCH_lvgl)/src/core/lv_refr.c
CY_IGNORE += $(SEARCH_lvgl)/demos
```

> ⚠️ The CY_IGNORE for `lv_draw_vg_lite.c`, `lv_draw_vg_lite_img.c`, `lv_vg_lite_utils.c`, and `lv_refr.c` are **critical** — Infineon provides custom overrides for these files (see §7). Without CY_IGNORE you get duplicate symbol errors.

---

## 4. Device Configurator — GFXSS Personality

**The single most important non-obvious requirement.** The GFXSS personality in `design.modus` generates:

| Symbol | Purpose |
|---|---|
| `GFXSS_DC_IRQ` | Display Controller interrupt |
| `GFXSS_GPU_IRQ` | GPU interrupt |
| `GFXSS_config` | GFXSS configuration structure |
| `GFXSS_GFXSS_MIPIDSI` | MIPI-DSI base address |
| `GFXSS_GFXSS_GPU_GCNANO` | GPU register base |
| `DISPLAY_I2C_CONTROLLER_HW` | I2C for touch controller |

**Without it:** Build fails with dozens of undefined symbol errors. No workaround with `#define`.

**How to get it:**
- **From LVGL template:** Already configured (no action needed)
- **Adding to existing project:** Copy entire `bsps/` directory from a working LVGL demo CE built for the same BSP, then rebuild

---

## 5. `lv_conf.h` — Critical Settings

Place in `proj_cm55/` root. Must be on include path before LVGL headers.

```c
#define LV_COLOR_DEPTH 16                           /* RGB565 for DSI */
#define LV_USE_OS   LV_OS_FREERTOS                  /* FreeRTOS integration */
#define LV_USE_STDLIB_MALLOC    LV_STDLIB_CLIB
#define LV_USE_STDLIB_STRING    LV_STDLIB_CLIB
#define LV_USE_STDLIB_SPRINTF   LV_STDLIB_CLIB

/* GPU acceleration */
#define LV_USE_DRAW_VG_LITE 1
#define LV_USE_DRAW_SW 1                            /* Software fallback */

/* GPU stride/buffer alignment — required for VG-Lite DMA */
#define LV_DRAW_BUF_STRIDE_ALIGN   64
#define LV_DRAW_BUF_ALIGN          64
#define LV_ATTRIBUTE_MEM_ALIGN_SIZE 64
#define LV_ATTRIBUTE_MEM_ALIGN      __attribute__((aligned(LV_ATTRIBUTE_MEM_ALIGN_SIZE)))

/* Memory placement */
#define LV_ATTRIBUTE_FAST_MEM       CY_SECTION(".cy_itcm")          /* ITCM for hot functions */
#define LV_ATTRIBUTE_LARGE_RAM_ARRAY CY_SECTION(".cy_socmem_data")  /* SoC shared memory */

/* Float + matrix for transforms */
#define LV_USE_FLOAT  1
#define LV_USE_MATRIX 1

/* Refresh: 18ms ≈ 55 FPS */
#define LV_DEF_REFR_PERIOD  18

/* Font strategy: enable only sizes you use (~20-60 KB each) */
#define LV_FONT_MONTSERRAT_14 1
#define LV_FONT_DEFAULT &lv_font_montserrat_14
```

> **Full file:** Copy `lv_conf.h` from the LVGL demo CE and modify. The file is ~800 lines.

---

## 6. Display Driver — `lv_port_disp.c`

```c
/* Framebuffers MUST be in .cy_gpu_buf section (GPU DMA accessible) */
CY_SECTION(".cy_gpu_buf") LV_ATTRIBUTE_MEM_ALIGN
    uint8_t disp_buf1[MY_DISP_HOR_RES * MY_DISP_VER_RES * BYTE_PER_PIXEL];
CY_SECTION(".cy_gpu_buf") LV_ATTRIBUTE_MEM_ALIGN
    uint8_t disp_buf2[MY_DISP_HOR_RES * MY_DISP_VER_RES * BYTE_PER_PIXEL];

static void disp_flush(lv_display_t *disp, const lv_area_t *area, uint8_t *color_p)
{
    vg_lite_finish();  /* CRITICAL: wait for GPU before swapping */
    Cy_GFXSS_Set_FrameBuffer((GFXSS_Type*) GFXSS, (uint32_t*) color_p, &gfx_context);
    if (ulTaskNotifyTake(pdTRUE, portMAX_DELAY))
        lv_display_flush_ready(disp);
}

void lv_port_disp_init(void)
{
    lv_display_t *disp = lv_display_create(MY_DISP_HOR_RES, MY_DISP_VER_RES);
    lv_display_set_flush_cb(disp, disp_flush);
    lv_tick_set_cb(xTaskGetTickCount);
    lv_display_set_buffers(disp, disp_buf1, disp_buf2, sizeof(disp_buf1),
                           LV_DISPLAY_RENDER_MODE_FULL);
    lv_display_set_resolution(disp, ACTUAL_DISP_HOR_RES, ACTUAL_DISP_VER_RES);
}
```

---

## 7. VG-Lite Override Files (MUST copy from Infineon LVGL demo)

Copy these 4 files from `mtb-example-psoc-edge-lvgl-demo` into `proj_cm55/`:

| File | Replaces |
|---|---|
| `lv_draw_vg_lite.c` | `lvgl/src/draw/vg_lite/lv_draw_vg_lite.c` |
| `lv_draw_vg_lite_img.c` | `lvgl/src/draw/vg_lite/lv_draw_vg_lite_img.c` |
| `lv_vg_lite_utils.c` | `lvgl/src/draw/vg_lite/lv_vg_lite_utils.c` |
| `lv_refr.c` | `lvgl/src/core/lv_refr.c` |

These contain Infineon-specific adaptations for the GFXSS hardware. **Do NOT write from scratch.**

---

## 8. Initialization Sequence (order matters)

### `main()` — before FreeRTOS scheduler

```
1. cybsp_init()
2. SCB_InvalidateDCache_by_Addr(0x240BD000, 0x43000)  ← Flush stale D-cache
3. __enable_irq()
4. xTaskCreate(cm55_gfx_task, "GFX", 8192, ...)
5. vTaskStartScheduler()
```

### `cm55_gfx_task()` — inside FreeRTOS context

```
1.  Configure GFXSS_config (MIPI, dimensions, framebuffer)
2.  Cy_GFXSS_Init()              ← Graphics Subsystem
3.  Register DC + GPU interrupts via NVIC
4.  Cy_SCB_I2C_Init()            ← I2C for touch
5.  vTaskDelay(500ms)            ← Let display stabilize
6.  Display panel init           ← mtb_disp_waveshare_4p3_init() etc.
7.  vg_lite_init_mem()           ← VGLite heap allocation
8.  vg_lite_init(W/4, H/4)      ← VGLite engine
9.  lv_init()                    ← LVGL core
10. lv_port_disp_init()          ← Display driver
11. lv_port_indev_init()         ← Touch input
12. your_ui_create()             ← Your UI
13. Loop: lv_timer_handler()     ← LVGL main loop (clamp delay ≥ 1ms)
```

---

## 9. Memory Budget

| Item | 4.3" (832×480) | 7"/10.1" (1024×600) |
|---|---|---|
| Framebuffer × 2 | ~1.6 MB | ~2.4 MB |
| VGLite heap | ~245 KB | ~245 KB |
| **Total .cy_gpu_buf** | **~1.8 MB** | **~2.65 MB** |
| FreeRTOS heap (min) | 50 KB | 50 KB |
| GFX task stack | 32 KB | 32 KB |

---

## 10. Top Pitfalls

1. **Missing GFXSS personality** → dozens of undefined symbols. See §4.
2. **Framebuffers not in `.cy_gpu_buf`** → black screen or GPU hard fault
3. **Missing CY_IGNORE for LVGL originals** → duplicate symbol errors
4. **4.3" uses 832px width** (not 800) for GPU stride alignment
5. **`vg_lite_finish()` missing before FB swap** → screen tearing
6. **FreeRTOS heap < 50 KB** → silent crash during `lv_init()`
7. **`lv_timer_handler()` returns 0** → clamp to 1ms or lower-priority tasks starve
8. **FT5406 touch coordinates inverted** → invert x/y in read callback
9. **`COMPONENTS+=GFXSS` missing from `common.mk`** → `cy_graphics.h` not found
10. **D-cache stale lines after `cybsp_init()`** → IPC/shared-memory corruption

## Verification Checklist

- [ ] `make getlibs` completes (all `.mtb` deps resolved)
- [ ] `make build` succeeds for all three cores
- [ ] Device Configurator shows GFXSS personality enabled
- [ ] 4 VG-Lite override files present in `proj_cm55/`
- [ ] `lv_conf.h` has `LV_COLOR_DEPTH 16`, `LV_USE_OS LV_OS_FREERTOS`, `LV_USE_DRAW_VG_LITE 1`
- [ ] Display shows UI content (not black, not garbled)
- [ ] Touch input activates correct UI elements
