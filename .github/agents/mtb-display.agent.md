---
name: mtb-display
description: Add LVGL v9 touchscreen graphics with VG-Lite GPU acceleration to a PSOC Edge E84 ModusToolbox project. Covers display drivers, touch input, Device Configurator GFXSS personality, lv_conf.h, framebuffer layout, and the full initialization sequence. Use for any graphics or display task.
tools: ["read", "edit", "create", "search", "shell"]
---

# Display & Graphics — LVGL v9 on PSOC Edge E84

You are an expert in integrating LVGL v9 touchscreen graphics with VG-Lite GPU acceleration on PSOC Edge E84.

> **CRITICAL prerequisite:** The project MUST have been created with `project-creator-cli` from an LVGL-capable template (see `mtb-project` agent). The Device Configurator GFXSS personality generates symbols that cannot be replicated manually.

## Strategy: Start from the Harder Capability

Start from the **Infineon LVGL demo CE** (`mtb-example-psoc-edge-lvgl-demo`) via `project-creator-cli`, then add your application logic on top. LVGL + GFXSS + VG-Lite requires BSP personalities, override files, and GPU memory sections that are difficult to retrofit. Other capabilities (WiFi, MQTT, BLE) are easier to add incrementally.

If adding LVGL to an existing project, copy the `bsps/` directory from a working LVGL demo built for the same BSP.

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
display-dsi-waveshare-4-3-lcd.mtb
display-dsi-waveshare-7-0-lcd-c.mtb
display-tft-ek79007ad3.mtb
touch-ctp-ft5406.mtb
touch-ctp-gt911.mtb
touch-ctp-ili2511.mtb
```

Each follows format: `https://github.com/Infineon/[name]#release-v1.0.0#$$ASSET_REPO$$/[name]/release-v1.0.0`

Run `make getlibs` after creating all `.mtb` files.

---

## 3. Makefile Configuration

### `common.mk` (workspace root)

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

> ⚠️ CY_IGNORE for `lv_draw_vg_lite.c`, `lv_draw_vg_lite_img.c`, `lv_vg_lite_utils.c`, and `lv_refr.c` are **critical** — Infineon provides custom overrides. Without CY_IGNORE → duplicate symbol errors.

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

**Without it:** Build fails with dozens of undefined symbol errors. No workaround.

**How to get it:**
- **From LVGL template:** Already configured
- **Adding to existing project:** Copy entire `bsps/` from a working LVGL demo CE

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
#define LV_ATTRIBUTE_FAST_MEM       CY_SECTION(".cy_itcm")
#define LV_ATTRIBUTE_LARGE_RAM_ARRAY CY_SECTION(".cy_socmem_data")

#define LV_USE_FLOAT  1
#define LV_USE_MATRIX 1
#define LV_DEF_REFR_PERIOD  18     /* ~55 FPS */

#define LV_FONT_MONTSERRAT_14 1
#define LV_FONT_DEFAULT &lv_font_montserrat_14
```

> **Full file:** Copy `lv_conf.h` from the LVGL demo CE and modify. ~800 lines total.

---

## 6. Display Driver — `lv_port_disp.c`

```c
/* Framebuffers MUST be in .cy_gpu_buf section */
CY_SECTION(".cy_gpu_buf") LV_ATTRIBUTE_MEM_ALIGN
    uint8_t disp_buf1[MY_DISP_HOR_RES * MY_DISP_VER_RES * BYTE_PER_PIXEL];
CY_SECTION(".cy_gpu_buf") LV_ATTRIBUTE_MEM_ALIGN
    uint8_t disp_buf2[MY_DISP_HOR_RES * MY_DISP_VER_RES * BYTE_PER_PIXEL];

static void disp_flush(lv_display_t *disp, const lv_area_t *area, uint8_t *color_p)
{
    vg_lite_finish();  /* Wait for GPU before swapping */
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

These contain Infineon-specific GFXSS adaptations. **Do NOT write from scratch.**

---

## 8. Initialization Sequence (order matters)

### `main()` — before FreeRTOS scheduler

```
1. cybsp_init()
2. SCB_InvalidateDCache_by_Addr(0x240BD000, 0x43000)
3. __enable_irq()
4. xTaskCreate(cm55_gfx_task, "GFX", 8192, ...)
5. vTaskStartScheduler()
```

### `cm55_gfx_task()` — inside FreeRTOS

```
1.  Configure GFXSS_config (MIPI, dimensions, framebuffer)
2.  Cy_GFXSS_Init()
3.  Register DC + GPU interrupts via NVIC
4.  Cy_SCB_I2C_Init()            ← Touch I2C
5.  vTaskDelay(500ms)            ← Let display stabilize
6.  Display panel init
7.  vg_lite_init_mem()           ← VGLite heap
8.  vg_lite_init(W/4, H/4)      ← VGLite engine
9.  lv_init()
10. lv_port_disp_init()
11. lv_port_indev_init()         ← Touch
12. your_ui_create()
13. Loop: lv_timer_handler()     ← Clamp delay ≥ 1ms
```

---

## 9. Memory Budget and Device Configurator Memory Configuration

### 9a. Display Memory Sizing

| Item | Formula | 4.3" (832×480) | 7"/10.1" (1024×600) |
|---|---|---|---|
| Framebuffer × 2 | `FB_W × FB_H × 2 (RGB565) × 2` | ~1.6 MB | ~2.4 MB |
| VGLite heap | `LV_VG_LITE_THORVG_BUF_ADDR_ALIGN` region | ~245 KB | ~245 KB |
| **Total gfx_mem needed** | FB + VGLite + headroom | **~2.0 MB min** | **~2.85 MB min** |
| FreeRTOS heap (min) | Depends on task count | 50 KB | 50 KB |
| GFX task stack | `lv_timer_handler` depth | 32 KB | 32 KB |
| LVGL working memory | Widgets, styles, draw buffers | 200–400 KB | 300–600 KB |

> **Framebuffer formula:** `width × height × bytes_per_pixel × num_buffers`
> For 4.3" DSI: `832 × 480 × 2 × 2 = 1,597,440 bytes` (~1.52 MB)
> VG-Lite GPU requires stride-aligned width (832, not 800) for DMA.

### 9b. PSOC Edge E84 Memory Architecture

The Device Configurator **Memory Configuration** personality controls how on-chip memory is partitioned between cores. Changes here generate linker scripts in `bsps/TARGET_*/config/GeneratedSource/`.

| Region | Total | Contains | Notes |
|---|---|---|---|
| **System SRAM** | 1 MB (`0x100000`) | `m33_code`, `m33_data` | CM33 code and data — often imbalanced in defaults |
| **SOCMEM** | 5 MB (`0x500000`) | `m55_code`, `m55_data`, `m55_data_secondary`, `gfx_mem`, `m33_m55_shared` | CM55 app + display buffers |
| **Flash** | 8 MB | Per-core flash regions | Rarely needs rebalancing |
| **DTCM** | 128 KB | CM55 fast data | Stack, critical vars |
| **ITCM** | 64 KB | CM55 fast code | `LV_ATTRIBUTE_FAST_MEM` target |

### 9c. gfx_mem Sizing (CRITICAL for Display Projects)

The `gfx_mem` region in SOCMEM holds framebuffers and VG-Lite GPU heap. Undersized = display corruption or GPU faults.

**Sizing formula:**
```
gfx_mem_bytes = (FB_WIDTH × FB_HEIGHT × 2 × NUM_BUFFERS) + VGLITE_HEAP_SIZE + HEADROOM

Example (4.3" display, 3 MB gfx_mem = 0x300000):
  Framebuffers: 832 × 480 × 2 × 2 = 1,597,440 bytes (1.52 MB)
  VG-Lite heap: 250,880 bytes (0.24 MB)
  Total used:   1,808 KB of 3,072 KB (59% — healthy headroom)
```

**Utilization target:** 50-75%. Below 50% = wasting SOCMEM that CM55 could use. Above 80% = risk of VG-Lite allocation failures under load.

**How to verify utilization:** Check linker map for `.cy_gpu_buf` section, or inspect at runtime:
```gdb
x/4xw &contiguous_mem       # VG-Lite heap base
print xPortGetFreeHeapSize() # FreeRTOS heap (separate from gfx_mem)
```

### 9d. Common Memory Imbalances and Fixes

| Symptom | Root Cause | Fix in Device Configurator |
|---|---|---|
| CM33 linker error: `.data` or `.bss` overflow | `m33_code` oversized, `m33_data` undersized | Shrink `m33_code` (128 KB is sufficient for connectivity-only CM33), grow `m33_data` |
| Display corruption, partial rendering | `gfx_mem` too small for 2× framebuffers + VG-Lite | Grow `gfx_mem` — use formula above; shrink `m55_data_secondary` |
| CM55 crash during LVGL init | `m55_data` or `m55_data_secondary` too small for LVGL heap | Grow `m55_data_secondary` — LVGL + widgets need 200-400 KB minimum |
| WiFi/MQTT stack crashes on CM33 | `m33_data` < 512 KB with WiFi + MQTT + TLS | CM33 with full WiFi stack needs ~530 KB data minimum |

### 9e. Device Configurator Memory Configuration Workflow

**To rebalance memory regions:**

1. Open `design.modus` in Device Configurator
2. Navigate to **Memory Configuration** (not Peripherals)
3. Regions are contiguous within each memory block — changing one size shifts neighbors
4. **Verify address continuity:** each region's start = previous region's start + size
5. Save → generates updated linker scripts in `GeneratedSource/`
6. Clean build required: `make clean && make build`

**Key rules:**
- Totals must match hardware: System SRAM = `0x100000`, SOCMEM = `0x500000`
- Regions are contiguous — no gaps allowed between regions in the same memory block
- Hex entry is easiest: calculate sizes in hex, verify with `start + size = next_start`
- Always rebuild after changes — linker scripts are generated artifacts

### 9f. Reference: Display Project Memory Layout (4.3" DSI, Dual-Core)

Proven layout from smart-appliance project (CM55 LVGL + CM33 WiFi/MQTT):

| Region | Start | Size (hex) | Size (KB) | Purpose |
|---|---|---|---|---|
| **System SRAM** | | | **1024** | |
| m33_code | `0x24000000` | `0x20000` | 128 | CM33 flash-resident; code is small |
| m33_data | `0x24020000` | `0x85000` | 532 | WiFi/MQTT/TLS heap, stacks, BSS |
| m33_ns_ipc | `0x240A5000` | `0x1000` | 4 | IPC driver |
| m33_reserved | — | — | — | BSP default (keep) |
| **SOCMEM** | | | **5120** | |
| m55_code | `0x24100000` | `0xC8000` | 800 | LVGL + app code (XIP or copy) |
| m55_data | `0x241C8000` | `0x80000` | 512 | LVGL heap, FreeRTOS heap, BSS |
| m55_data_secondary | `0x24248000` | `0x198000` | 1632 | Overflow data, large arrays |
| gfx_mem | `0x243E0000` | `0x300000` | 3072 | 2× framebuffers + VG-Lite heap |
| m33_m55_shared | `0x246E0000` | `0x8000` | 32 | Lock-free IPC struct |

> **This layout is a starting point.** Adjust `gfx_mem` and `m55_data_secondary` based on display size and application complexity. Always verify totals equal hardware capacity.

---

## 10. VG-Lite Performance — Avoiding Flicker and Frame Drops

### The Layer Compositing Trap (CRITICAL)

**When a container/parent object has `opa < LV_OPA_COVER` (255) and has children, LVGL allocates a temporary layer buffer, renders all children into it, then composites the layer onto the framebuffer at the target opacity — EVERY FRAME the object is dirty.**

This is the #1 cause of animation flicker on PSOC Edge E84.

**Bad pattern — container opacity animation (forces layer compositing):**
```c
/* DON'T: animating opa on a group with children */
lv_obj_set_style_opa(group, LV_OPA_TRANSP, LV_PART_MAIN);
lv_anim_set_exec_cb(&a, set_opa);  /* changes group opa → layer alloc per frame */
```

**Good pattern — per-child opacity animation (no layers needed):**
```c
/* DO: animate opa on each child individually (no children → no layer) */
static void set_children_opa(void *obj, int32_t v) {
    uint32_t cnt = lv_obj_get_child_count((lv_obj_t *)obj);
    for (uint32_t i = 0; i < cnt; i++) {
        lv_obj_set_style_opa(lv_obj_get_child((lv_obj_t *)obj, i), v, LV_PART_MAIN);
    }
}
/* Container stays fully opaque, children get per-element opa */
lv_anim_set_exec_cb(&a, set_children_opa);
```

### What Gets GPU-Accelerated vs. Software Fallback

The PSOC Edge E84 GCNANO (ChipID 0x265) has a preference score of 80/255 in LVGL's draw dispatch.

| Render Type | GPU? | Notes |
|---|---|---|
| FILL (solid, gradient) | ✅ Yes | Linear gradients only (no radial) |
| BORDER | ✅ Yes | |
| IMAGE (RGB565, ARGB8888) | ✅ Yes | Format-dependent |
| ARC, LINE, TRIANGLE | ✅ Yes | `line_rounded` may fall back to SW |
| BOX_SHADOW | ✅ if enabled | Set `LV_VG_LITE_USE_BOX_SHADOW 1` in lv_conf.h |
| **LABEL (text)** | ❌ Always SW | No font→GPU path exists in LVGL |
| Radial gradient | ❌ No | Hardware doesn't support it |
| Gaussian blur | ❌ No | Hardware doesn't support it |

### Animation Performance Rules

1. **Never animate opacity on containers with children** — use per-child opa or pre-rendered layers
2. **Text (labels) are always SW-rendered** — avoid high-frequency opacity animations on text objects (causes constant SW re-rendering)
3. **`translate_x`/`translate_y` animations** invalidate both old and new positions — minimize the moving area
4. **`LV_VG_LITE_FLUSH_MAX_COUNT = 0`** enables CPU/GPU parallelism — critical for text-heavy UIs where SW and GPU rendering overlap
5. **Keep concurrent animations low** — 4-5 simultaneous animations is the practical limit before frame drops
6. **`shadow_width` on objects with text children** forces SW fallback — use `border_width` + `border_opa` instead for glow effects
7. **Prefer LVGL drawing primitives over bitmap assets for animated elements** — arcs, lines, circles, and shapes are GPU-native (VG-Lite renders them directly), resolution-independent, zero asset management, and easier to animate (change draw parameters, not swap bitmaps). Reserve bitmap images for static content like hero images and photos

### Recommended `lv_conf.h` Tuning (Performance)

```c
/* GPU/CPU parallelism — 0 means flush immediately, letting CPU do SW work while GPU renders */
#define LV_VG_LITE_FLUSH_MAX_COUNT  0

/* GPU-accelerated box shadows (layered borders, not true shadows) */
#define LV_VG_LITE_USE_BOX_SHADOW   1

/* Right-size GPU caches to save memory — defaults are 32 each */
#define LV_VG_LITE_GRAD_CACHE_CNT   4   /* Increase if using many gradients */
#define LV_VG_LITE_STROKE_CACHE_CNT 8   /* Increase if using many stroked paths */
```

### GCNANO 0x265 Hardware Capabilities (12 of 45 VG-Lite features)

**Supported:** IM_INDEX_FORMAT, SCISSOR, BORDER_CULLING, RGBA2_FORMAT, DOUBLE_IMAGE, YUV_OUTPUT, IM_INPUT, SRC_PREMULTIPLIED, PARALLEL_PATHS, STRIPE_MODE, YUY2_INPUT, 16PIXELS_ALIGN

**NOT supported (key gaps):** QUALITY_8X (max 4x AA), RADIAL_GRADIENT, GLOBAL_ALPHA, GAUSSIAN_BLUR, COLOR_KEY, MASK, MIRROR, GAMMA, DITHER, COLOR_TRANSFORMATION

GPU fill rate: 200 Mpixels/s | Max display: 1024×768@60Hz | 8 Porter-Duff blend modes

## 11. Top Pitfalls

1. **Missing GFXSS personality** → dozens of undefined symbols (§4)
2. **Framebuffers not in `.cy_gpu_buf`** → black screen or GPU hard fault
3. **Missing CY_IGNORE for LVGL originals** → duplicate symbol errors
4. **4.3" uses 832px width** (not 800) for GPU stride alignment
5. **`vg_lite_finish()` missing before FB swap** → screen tearing
6. **FreeRTOS heap < 50 KB** → silent crash during `lv_init()`
7. **`lv_timer_handler()` returns 0** → clamp to 1ms or tasks starve
8. **FT5406 touch inverted** → invert x/y in read callback
9. **`COMPONENTS+=GFXSS` missing from `common.mk`** → `cy_graphics.h` not found
10. **D-cache stale lines after `cybsp_init()`** → IPC/shared-memory corruption
11. **Animating opacity on containers with children** → layer compositing every frame → flicker (§10)

## Verification Checklist

- [ ] `make getlibs` completes (all `.mtb` deps resolved)
- [ ] `make build` succeeds for all three cores
- [ ] Device Configurator shows GFXSS personality enabled
- [ ] Memory Configuration: `gfx_mem` ≥ 2× framebuffer + VG-Lite heap (use formula in §9c)
- [ ] Memory Configuration: region totals = System SRAM `0x100000` + SOCMEM `0x500000`
- [ ] Memory Configuration: all regions contiguous (no address gaps)
- [ ] 4 VG-Lite override files present in `proj_cm55/`
- [ ] `lv_conf.h` has `LV_COLOR_DEPTH 16`, `LV_USE_OS LV_OS_FREERTOS`, `LV_USE_DRAW_VG_LITE 1`
- [ ] Display shows UI content (not black, not garbled)
- [ ] Touch input activates correct UI elements
