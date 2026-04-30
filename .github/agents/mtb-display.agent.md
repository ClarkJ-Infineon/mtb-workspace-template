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

> **Validated:** The Matter Thermostat PoC confirmed this strategy. Adding graphics to an existing Matter project required 13+ integration steps and deep knowledge of GFXSS, MPU, and memory layout. Starting from the LVGL demo and adding Matter/WiFi on CM33 is significantly faster and less error-prone. See `reference/guides/matter-thermostat-poc-lessons-learned.md`.

If adding LVGL to an existing project, see §10b "Adding Graphics to an Existing Project" for the complete workflow.

---

## 1. Display Hardware Options

| Display | Size | Resolution | FB Res | Driver Library | Touch Library | `CONFIG_DISPLAY` | Connector | Rework |
|---|---|---|---|---|---|---|---|---|
| Waveshare 4.3" DSI | 4.3" | 800×480 | **832×480** ¹ | `display-dsi-waveshare-4-3-lcd` | `touch-ctp-ft5406` | `W4P3INCH_DISP` | J39 (FPC) | None |
| Waveshare 7" DSI (C) | 7" | 1024×600 | 1024×600 | `display-dsi-waveshare-7-0-lcd-c` | `touch-ctp-gt911` | `WS7P0DSI_RPI_DISP` | J39 + J41 (I2C) | None |
| Winstar 10.1" TFT | 10.1" | 1024×600 | 1024×600 | `display-tft-ek79007ad3` | `touch-ctp-ili2511` | `WF101JTYAHMNB0_DISP` | J38 + J37 | Remove R22-R27, populate R28-R33 |
| Dastek 1.43" AMOLED | 1.43" | 466×466 | **512×466** ² | `display-amoled-co5300` | `touch-ctp-ft6146-m00` | `CO5300_DISP` | J38 + J37 | Remove R22-R27+R462, populate R28-R33+R463 |
| Microtek 1.43" AMOLED | 1.43" | 466×466 | **512×466** ² | `display-amoled-co5300` | `touch-ctp-ft3268` | `CO5300_DISP` | J38 + J37 | Remove R22-R27+R462, populate R28-R33+R463 |

> ¹ 4.3" display: VG-Lite GPU requires stride alignment — 800px padded to 832px at RGB565. LVGL display created at 832×480, visible area clamped to 800×480 via `lv_display_set_resolution()`.
>
> ² 1.43" AMOLED: 466px padded to 512px for 128-byte stride alignment at RGB565 (`466 × 2 = 932`, not 128-byte aligned; `512 × 2 = 1024`). These are **command mode** displays with built-in GRAM — use `Cy_GFXSS_Transfer_Frame()` instead of DC continuous scan.

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
display-amoled-co5300.mtb
touch-ctp-ft5406.mtb
touch-ctp-gt911.mtb
touch-ctp-ili2511.mtb
touch-ctp-ft6146-m00.mtb
touch-ctp-ft3268.mtb
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

### GFXSS Personality Parameters (Device Configurator)

| Section | Parameter | Description |
|---|---|---|
| **Clocks** | Root clock for graphics | Source clock feeding GPU + DC (CLK_HF1) |
| **Clocks** | Root clock for MIPI DPHY PLL | Separate clock for DSI PHY (CLK_HF12, typically 24 MHz) |
| **General** | Display Type | DSI Video, DSI Command, DBI, etc. |
| **General** | Transfer Type | Burst, Non-burst sync pulses, Non-burst sync events |
| **General** | Width / Height | Display resolution in pixels |
| **General** | Target FPS | Frame rate (affects clock calculations) |
| **General** | GPU Enable | **Must be ON** for VG-Lite acceleration |
| **General** | HSYNC / VSYNC | Sync signal widths |
| **General** | HBP / HFP / VBP / VFP | Display timing parameters (from display datasheet) |
| **Layers** | Graphics/Video Layer | Primary layer — always enabled |
| **Layers** | Overlay0 / Overlay1 | Optional overlays composited via per-pixel alpha blending |
| **Layer config** | Input Format | RGB565, ARGB8888, etc. per layer |
| **Layer config** | Position (X,Y start/end) | Layer window coordinates |

**Display timing reference values:**

| Display | HSYNC | HBP | HFP | VSYNC | VBP | VFP | Pixel Clock |
|---|---|---|---|---|---|---|---|
| Waveshare 4.3" | 10 | 20 | 210 | 5 | 20 | 20 | 33.768 MHz |
| 1.43" AMOLED | — | — | — | — | — | — | Command mode (DBI) |

**DC layer architecture:** 3 compositing layers — Graphics/Video (base) + Overlay0 + Overlay1. Overlay1 does NOT support tiled mode. RGB formats use linear tiling; YUV formats use tiled.

**Max display resolution:** 1024 × 768 @ 60 Hz (64 MHz pixel clock), 24-bit color.

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

/* GPU stride/buffer alignment — 128B required for DC framebuffer, 64B min for GPU */
#define LV_DRAW_BUF_STRIDE_ALIGN   128
#define LV_DRAW_BUF_ALIGN          128
#define LV_ATTRIBUTE_MEM_ALIGN_SIZE 128
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
    vg_lite_finish();  /* Wait for GPU to complete rendering */

    /* CRITICAL: Drain stale DC interrupt notification.
     * The DC fires end-of-frame (VSYNC) interrupts continuously while the display
     * is scanning. If a VSYNC fires during GPU rendering (normal — display keeps
     * scanning while GPU works), a task notification is queued. Without draining it,
     * the ulTaskNotifyTake() below returns immediately with the STALE notification
     * instead of waiting for a FRESH VSYNC after the buffer swap.
     * Result without drain: buffer displayed before DC latches new address → tearing. */
    ulTaskNotifyTake(pdTRUE, 0);  /* Non-blocking drain — discard stale notification */

    Cy_GFXSS_Set_FrameBuffer((GFXSS_Type*) GFXSS, (uint32_t*) color_p, &gfx_context);

    /* Wait for fresh VSYNC confirming the DC has latched the new buffer address */
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

Proven layout for a dual-core project (CM55 LVGL + CM33 WiFi/MQTT):

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
6. **`shadow_width > 0` on ANY animated object is expensive** — even without text children, shadow rendering at 30fps causes frame drops. Use `bg_color` + `bg_opa` on a larger circle/rect instead. Shadows are only acceptable on static (non-animated) objects.
7. **Prefer LVGL drawing primitives over bitmap assets for animated elements** — arcs, lines, circles, and shapes are GPU-native (VG-Lite renders them directly), resolution-independent, zero asset management, and easier to animate (change draw parameters, not swap bitmaps). Reserve bitmap images for static content like hero images and photos
8. **Guard periodic refresh callbacks with state caching** — `lv_obj_set_style_*()` ALWAYS marks the object dirty (triggers full-frame re-render in RENDER_MODE_FULL), even if the value hasn't changed. Always cache previous state and skip updates when unchanged (see "Conditional Widget Update Pattern" below)

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

### Conditional Widget Update Pattern (Preventing Unnecessary Redraws)

In `RENDER_MODE_FULL`, ANY dirty widget triggers a complete 800×480 GPU render. Periodic refresh callbacks (e.g., every 500ms to update sensor values) must NOT unconditionally call `lv_label_set_text()` or `lv_obj_set_style_*()` — these always invalidate.

**Helper — only update label if text changed:**
```c
static inline void label_set_if_changed(lv_obj_t *label, const char *new_text)
{
    const char *cur = lv_label_get_text(label);
    if (cur == NULL || strcmp(cur, new_text) != 0) {
        lv_label_set_text(label, new_text);
    }
}
```

**State caching pattern for refresh callbacks:**
```c
static int16_t  s_prev_temp  = INT16_MIN;
static uint8_t  s_prev_mode  = UINT8_MAX;
static uint16_t s_prev_timer = UINT16_MAX;

static void page_refresh_cb(lv_timer_t *timer)
{
    /* Only update widgets when underlying data actually changed */
    if (g_app_state.temperature != s_prev_temp) {
        s_prev_temp = g_app_state.temperature;
        char buf[16];
        lv_snprintf(buf, sizeof(buf), "%d°C", s_prev_temp);
        lv_label_set_text(lbl_temp, buf);
    }
    if (g_app_state.mode != s_prev_mode) {
        s_prev_mode = g_app_state.mode;
        lv_obj_set_style_bg_color(btn_mode, mode_colors[s_prev_mode], LV_PART_MAIN);
    }
    /* ... same pattern for all widgets */
}
```

> **Rule:** If a value hasn't changed, skip the widget update entirely. This prevents spurious dirty-marking and eliminates unnecessary full-frame renders during idle periods.

### GCNANO 0x265 Hardware Capabilities (12 of 45 VG-Lite features)

**Supported:** IM_INDEX_FORMAT, SCISSOR, BORDER_CULLING, RGBA2_FORMAT, DOUBLE_IMAGE, YUV_OUTPUT, IM_INPUT, SRC_PREMULTIPLIED, PARALLEL_PATHS, STRIPE_MODE, YUY2_INPUT, 16PIXELS_ALIGN

**NOT supported (key gaps):** QUALITY_8X (max 4x AA), RADIAL_GRADIENT, GLOBAL_ALPHA, GAUSSIAN_BLUR, COLOR_KEY, MASK, MIRROR, GAMMA, DITHER, COLOR_TRANSFORMATION

GPU fill rate: 200 Mpixels/s | Max display: 1024×768@60Hz | 8 Porter-Duff blend modes

### VG-Lite API Quick Reference

**Initialization & lifecycle:**
- `vg_lite_set_command_buffer_size(size)` — call BEFORE `vg_lite_init` (default 64 KB)
- `vg_lite_init(tess_w, tess_h)` — tessellation window multiples of 16; min 16×16; pass (0,0) for blit-only
- `vg_lite_init_mem(param)` — set GPU register base and heap region
- `vg_lite_close()` — MUST call before re-initializing; single-thread only

**Frame submission:**
- `vg_lite_flush()` — async submit (GPU works while CPU prepares next draw) — **use in hot path**
- `vg_lite_finish()` — sync wait (blocks CPU) — use only at frame end or when results needed

**Drawing:**
- `vg_lite_clear(target, rect, color)` — fast fill (NULL rect = entire buffer)
- `vg_lite_draw(target, path, fill_rule, matrix, blend, color)` — path fill (affine only, no perspective)
- `vg_lite_draw_grad(target, path, fill_rule, matrix, grad, blend)` — linear gradient fill
- `vg_lite_draw_pattern(target, path, fill, path_matrix, pattern, pat_matrix, blend, mode, color, mix, filter)` — pattern fill
- `vg_lite_blit(target, source, matrix, blend, color, filter)` — image compositing
- `vg_lite_blit_rect(target, source, rect, matrix, blend, color, filter)` — sub-region blit

**Paths:**
- `vg_lite_init_path(...)` — standard path (S8/S16/S32/FP32 coordinates)
- `vg_lite_init_arc_path(...)` — arc path (**must use FP32 format**)
- `vg_lite_upload_path(path)` — upload static path to GPU memory (fonts, icons, repeated elements)
- Arc opcodes (0x13–0x1A): `SCCWARC`, `SCWARC`, `LCCWARC`, `LCWARC` with args (rh, rv, rotation, x, y)

**Strokes:**
- `vg_lite_set_stroke(path, cap, join, width, miter, dash, count, phase, color)` — cap: BUTT/ROUND/SQUARE
- `vg_lite_update_stroke(path)` — generate stroke path data (call after set_stroke, before draw)
- Stroke line_width < 0.5 silently prevents stroking

**Gradients:**
- `vg_lite_set_grad(grad, count, colors, stops)` → `vg_lite_update_grad(grad)` → `vg_lite_draw_grad(...)`
- Up to **16 color stops** (VLC_MAX_GRADIENT_STOPS); stops range 0–255
- **⚠️ Radial gradients NOT supported on GCNanoUltraV** — `vg_lite_draw_radial_grad` returns NOT_SUPPORT
- **Workaround:** Pre-render radial gradient as image → use `vg_lite_draw_pattern` with PATTERN_REPEAT

**Scissor (legacy — better performance than mask layer):**
- `vg_lite_set_scissor(x, y, right, bottom)` — enable scissor clip
- Pass `(-1, -1, -1, -1)` to disable

**Blend modes (Porter-Duff):**
- Standard: `NONE`, `SRC_OVER`, `DST_OVER`, `SRC_IN`, `DST_IN`, `MULTIPLY`, `SCREEN`, `ADDITIVE`, `SUBTRACT`
- LVGL-specific: `NORMAL_LVGL`, `ADDITIVE_LVGL`, `SUBTRACT_LVGL`, `MULTIPLY_LVGL` — use SW+HW hybrid on GCNanoUltraV
- All Porter-Duff modes assume **pre-multiplied alpha** (except ADDITIVE and *_LVGL modes)

**Filter modes (image blits):**
- `POINT` — nearest neighbor, fastest
- `LINEAR` — horizontal interpolation
- `BI_LINEAR` — 2×2 box, good quality
- `GAUSSIAN` — 3×3 convolution, most expensive

**Tessellation buffer sizing:**
- Match to most common path bounding box (e.g., 24×24 for 24pt fonts)
- Undersized = path resubmitted multiple times → performance cliff
- Width and height must be multiples of 16; minimum 16×16

**Threading:** VGLite V3 driver is **single-thread only**. No concurrent contexts. To switch threads: `vg_lite_close()` → `vg_lite_init()` in new thread.

### Unsupported VG-Lite APIs on GCNanoUltraV (DO NOT USE)

These compile but return `VG_LITE_NOT_SUPPORT` at runtime:

| Category | Unsupported Functions |
|---|---|
| Radial gradient | `vg_lite_draw_radial_grad`, `set/update/get/clear_radial_grad` |
| Extended linear gradient | `vg_lite_draw_linear_grad`, `set/update/get/clear_linear_grad` |
| Color transform | `enable/disable/set_color_transform` |
| Mask layers | `enable/disable/create/fill/blend/set/render/destroy_mask_layer` |
| Global alpha | `source_global_alpha`, `dest_global_alpha` |
| Pixel effects | `enable/disable_dither`, `set_gamma`, `gaussian_filter`, `set_color_key` |
| Other | `set_pixel_matrix`, `enable/disable_scissor` (use `set_scissor` instead) |

### VG-Lite Color Format: `vg_lite_color_t` is ABGR

```
Bits:  31:24  23:16  15:8   7:0
       Alpha  Blue   Green  Red
```

**Gotcha:** Gradient colors in `vg_lite_set_grad()` use ARGB8888 with alpha in upper byte — different byte order from `vg_lite_color_t`.

### Power States for Display Applications

| State | GPU | CPU Clock | Display | Frame Rate | Transition |
|---|---|---|---|---|---|
| **HP** (High Performance) | Enabled, 200 MHz | CM55 @ 400 MHz | Active, full refresh | 30-36 fps (4.3"), 22-24 fps (1.43") | Default state |
| **LP** (Low Power) | Disabled (clock-gated) | CM55 @ 50-140 MHz | Active, reduced refresh | ~1 fps | `vg_lite_close` → `GFXSS_DeInit` → clock switch → `GFXSS_Init` (GPU disabled) |
| **ULP** (Ultra-Low Power) | Disabled | Deep sleep | OFF, DSI ULPM | 0 | Display OFF → `Cy_MIPIDSI_EnterULPM` → DeepSleep |

**HP → LP transition pattern:**
```c
vg_lite_close();
Cy_GFXSS_Disable_GPU_Interrupt(base);
Cy_GFXSS_DeInit(base, &gfx_context);
// Switch to LP/ULP clocks
GFXSS_config.gpu_cfg->enable = false;
Cy_GFXSS_Init(base, &GFXSS_config, &gfx_context);
```

**LP display refresh:** Set LVGL refresh timer to 1000 ms (video mode) or 9000 ms (command mode):
```c
lv_timer_set_period(lv_display_get_refr_timer(disp), LVGL_REFRESH_TIME_MS);
lv_timer_set_period(lv_anim_get_timer(), LVGL_REFRESH_TIME_MS);
```

### RLAD Image Compression

DC hardware includes an RLAD (Run-Length Adaptive Dithering) decoder for real-time decompressed display:
- ~33% compression ratio on typical UI assets
- Encoding done **offline** with RLAD encoder tool (distributed with CE239203)
- Decoding on-the-fly by DC hardware — **one layer only**
- Supports: RGB565, ARGB8888, RGB888, RGB666, ARGB4444, ARGB1555, GRAY4/6/8
- Use for static backgrounds, splash screens, pre-rendered assets
- See CE239203 for implementation reference

### Display Brightness Control

- **Video mode displays:** Use display driver library brightness API (display-specific)
- **Command mode displays (AMOLED):** Send DCS brightness commands via `mtb_display_co5300_*` APIs
- Brightness reduction is a key LP power savings lever

## 10b. Adding Graphics to an Existing Project

> Use this workflow when adding display/LVGL to a project that already has WiFi, MQTT, BLE, Matter, or other non-graphics functionality. Starting from a graphics template and adding connectivity is preferred (§Strategy above), but sometimes the existing project is too complex to recreate.
>
> **Validated:** The Matter Thermostat PoC completed this workflow over 22 checkpoints. It works but is significantly harder than the reverse path. Budget 2–3 days of integration and debugging.

### Prerequisites

Before starting, you need a **reference LVGL project** for the same BSP and display:
- `mtb-example-psoc-edge-lvgl-demo` — official Infineon code example (available via Eclipse IDE for ModusToolbox™ New Application wizard or GitHub)

You will copy hardware driver files, VG-Lite overrides, and LVGL configuration from this reference. **Do not attempt to create these from scratch.** The reference project's `design.modus` serves as a visual guide for what to configure in Device Configurator — you will replicate the settings manually, not copy the file.

### Step-by-Step Workflow

**1. Configure GFXSS personality in Device Configurator**

Open your existing project's `design.modus` in Device Configurator. Using the reference LVGL project's `design.modus` as a visual guide (open it in a second Device Configurator window), manually add the following settings to your existing configuration:

- **GFXSS personality:** Enable and configure — display type (DSI), resolution, timing parameters, layer count, GPU enable. Match all values from the reference project.
- **I2C controller for touch:** Enable SCB0 (or appropriate SCB) with correct SCL/SDA pin assignments for your display's touch controller.
- **Clocks:** CLK_HF1 = 400 MHz (GPU+CM55), CLK_HF12 (MIPI DPHY reference clock). Verify these don't conflict with existing clock assignments.
- **Protection settings (proj_cm33_s):** GFXSS peripheral must be accessible from CM55. Check the Protection Configuration panel and grant CM55 access.

> **IMPORTANT:** Do NOT replace your entire `design.modus` with the reference project's file — you would lose all existing peripheral configurations (WiFi, BLE, sensors, etc.). Add the GFXSS-related settings alongside your existing ones.

**Save → Close → Reopen → Verify** settings persisted. Delete `design.modus.lock` if changes don't take effect.

**2. Memory Configurator changes (design.modus → Memory Configuration)**
- Add `gfx_mem` region in SOCMEM (minimum 2 MB for 4.3", 3 MB recommended — see §9c for formula)
- Shrink `m55_data_secondary` to make room (**shrink first, then grow** — tool validates intermediate states)
- If CM33 runs WiFi/MQTT/TLS: ensure `m33_data` ≥ 530 KB
- Verify region address continuity and totals: System SRAM = `0x100000`, SOCMEM = `0x500000`

**3. MPU Configuration (Device Configurator → MPU panel)**
- Configure MPU non-cacheable region for `gfx_mem` on CM55 — match the address and size from step 2
- Configure MPU non-cacheable region for IPC shared memory (`m33_m55_shared`) on BOTH cores
- **Verify `cycfg_system.c` MPU regions match `cymem_CM55_0.h` defines** — MPU does NOT auto-sync with Memory Configurator

**4. Regenerate code**

After steps 1-3, regenerate Device Configurator output:
- Save `design.modus` in Device Configurator
- Or run: `make build` (triggers code generation automatically)
- Verify generated files in `bsps/TARGET_*/config/GeneratedSource/` are updated (check timestamps)

**5. CM55 Makefile additions**
```makefile
# In common.mk (MUST be common.mk, not per-project Makefile):
CONFIG_DISPLAY = W4P3INCH_DISP
COMPONENTS += GFXSS

# In proj_cm55/Makefile:
COMPONENTS += FREERTOS RTOS_AWARE
DEFINES += CY_RETARGET_IO_CONVERT_LF_TO_CRLF _BAREMETAL=0 CY_RTOS_AWARE
DEFINES += configENABLE_MVE=1    # CRITICAL for Helium/MVE SIMD (VG-Lite uses it)
# Add all CY_IGNORE lines for unused display/touch drivers (see §3 above)
# Add display-conditional driver selection from §3 above
```

**6. Create CM55 FreeRTOSConfig.h**

If CM55 previously had a stub or minimal `FreeRTOSConfig.h`, replace it with graphics-appropriate values:

```c
/* proj_cm55/FreeRTOSConfig.h — CRITICAL values for graphics */
#define configMINIMAL_STACK_SIZE        512     /* 32 KB — was likely 128 */
#define configTOTAL_HEAP_SIZE           (50 * 1024)  /* 50 KB minimum */
#define configUSE_TICKLESS_IDLE         0       /* Must be 0 for display refresh */
#define configENABLE_MVE                1       /* Helium SIMD for VG-Lite */
```

Copy remaining config values from the reference LVGL project's `FreeRTOSConfig.h`.

**7. Add dependency .mtb files to proj_cm55/deps/**
- LVGL, display driver, touch driver, FreeRTOS, retarget-io (see §2 for exact URLs)
- Run `make getlibs` to download all resolved dependencies

**8. Copy hardware driver files to proj_cm55/**

Copy from the reference LVGL project into `proj_cm55/`:

| Files | Purpose | Source in reference project |
|---|---|---|
| 4 VG-Lite override files | Infineon GFXSS adaptations (§7) | `proj_cm55/` root |
| `lv_port_disp.c` / `.h` | Display port (framebuffer swap, DC flush) | `proj_cm55/` |
| `lv_port_indev.c` / `.h` | Touch input port (FT5406/GT911 read) | `proj_cm55/` |
| `lv_conf.h` | LVGL configuration | `proj_cm55/` root |
| Display init wrappers | `mtb_disp_*` init/enable calls | `proj_cm55/` |
| Touch init wrapper | `mtb_touch_*` init call | `proj_cm55/` |

> **Count:** Expect 10-15 files total. The exact set varies by display. Copy ALL `.c`/`.h` files from the reference `proj_cm55/` that are not part of the reference project's application logic (i.e., exclude the demo's UI code, keep the driver/init infrastructure).

**9. Create retarget_io_init.c on CM55**

CM55 **MUST** call `init_retarget_io()` even if CM33 already uses the same UART. On PSOC Edge, this requires the 3-step PDL pattern (see Part 2 of `mtb-diagnostics` agent):

```c
/* proj_cm55/retarget_io_init.c */
#include "cybsp.h"
#include "mtb_hal.h"
#include "cy_retarget_io.h"

static cy_stc_scb_uart_context_t    DEBUG_UART_context;
static mtb_hal_uart_t               DEBUG_UART_hal_obj;

void init_retarget_io(void) {
    Cy_SCB_UART_Init(CYBSP_DEBUG_UART_HW, &CYBSP_DEBUG_UART_config, &DEBUG_UART_context);
    Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);
    mtb_hal_uart_setup(&DEBUG_UART_hal_obj, &CYBSP_DEBUG_UART_hal_config, &DEBUG_UART_context, NULL);
    cy_retarget_io_init(&DEBUG_UART_hal_obj);
}
```

Without this, any `printf()` on CM55 triggers `CY_HALT()` due to uninitialized mutex assertion.

**10. D-Cache invalidation after cybsp_init() (CRITICAL)**

In CM55 `main.c`, immediately after `cybsp_init()`:

```c
cybsp_init();
/* Flush stale D-Cache lines created before MPU was configured */
SCB_InvalidateDCache_by_Addr((void *)IPC_SHARED_ADDR, sizeof(ipc_shared_t));
init_retarget_io();
```

See `mtb-multicore` agent Part 1 "Boot-Time D-Cache Invalidation" for explanation.

**11. Resolve SCB (I2C bus) ownership conflicts**

If CM33 uses any I2C sensor on the same SCB as the display touch controller (typically SCB0), you must migrate or disable it. Only ONE core may own a given SCB.

**To migrate a sensor from CM33 to CM55:**
1. Remove sensor task/init from CM33 code (but keep its defines if the source still compiles)
2. Add sensor driver to CM55 — initialize on the same SCB after display/touch I2C init
3. Publish sensor readings via IPC shared memory back to CM33
4. On CM33: add sensor stale-timeout (if CM55 stops publishing, use last-known value)

> **I2C SDA stuck LOW:** If the display or sensor leaves SDA low at boot (common after power glitches), the I2C init will fail. Implement bus recovery by toggling SCL as GPIO (9 clocks) before I2C init. SDA usually releases after 1-2 clock pulses.

**12. Boot sequence changes**
- CM33 `main.c` must call `Cy_SysEnableCM55()` (if not already)
- If CM33 runs WiFi: it must NOT enter DeepSleep — keep FreeRTOS scheduler running
- Boot order: CM33 boots → enables CM55 → CM55 initializes graphics on FreeRTOS task

**13. Build and verify**
```bash
make clean && make -j8 build    # Full rebuild — all three cores
```
- Verify CM33 builds with no new errors (existing app unaffected)
- Verify CM55 builds with no undefined symbols
- Flash and check: display shows content, touch responds, no garbled output

**8. Common pitfalls specific to this workflow**
- **GFXSS not accessible from CM55:** `proj_cm33_s` protection settings must grant CM55 access to GFXSS peripheral registers. Symptom: HardFault on `Cy_GFXSS_Init`.
- **Memory layout mismatch:** If you copy `design.modus` but not the Memory Configuration, linker scripts won't include `gfx_mem`. Symptom: framebuffer placed in wrong memory, black screen.
- **Missing COMPONENTS+=GFXSS:** `cy_graphics.h` not found → dozens of missing header errors. **Must be in `common.mk`**, not per-project Makefile — Device Configurator generates `cy_graphics.h` includes in `cycfg_peripherals.c` compiled by ALL projects.
- **Clock configuration conflict:** If existing project sets CLK_HF1 to a non-400 MHz value, GPU performance degrades or DSI link fails. Verify clock tree after merging configurations.
- **Duplicate FreeRTOS config:** Both projects may define `FreeRTOSConfig.h` — ensure CM55 config has adequate heap and stack for graphics task (8192+ words stack).
- **FreeRTOS stack too small:** Graphics task needs `configMINIMAL_STACK_SIZE = 512` (32 KB stack). Existing non-graphics projects typically default to 128. Always check `FreeRTOSConfig.h` when adding LVGL to an existing project.
- **Missing `configENABLE_MVE=1`:** VG-Lite GPU may use Helium/MVE SIMD instructions. Without this define, FreeRTOS won't save MVE registers on context switch → silent data corruption. Add to CM55 Makefile DEFINES.
- **retarget-io not initialized on CM55 (CRITICAL):** ALWAYS call `init_retarget_io()` on CM55 even when CM33 uses the same UART (SCB2). The retarget-io library overrides `_write()` with a mutex-guarded version. Without init, `cy_retarget_io_mutex_initialized` assertion fails → `CY_HALT()`. Both cores sharing SCB2 is safe (output interleaves at byte level).
- **I2C bus ownership conflict:** Only ONE core may own a given SCB. Display/touch use SCB0. If CM33 uses SCB0 for a sensor, it must be disabled or migrated to CM55 before adding graphics.
- **I2C SDA stuck LOW at boot:** If a previous boot cycle or power glitch leaves SDA low, the I2C init fails with `CY_SCB_I2C_BAD_PARAM` or ADDR_NAK. Implement I2C bus recovery: configure SCL as GPIO, toggle 9 clock cycles, then reconfigure as I2C. SDA usually releases after 1-2 pulses.
- **Device Configurator lock file:** A `.lock` file prevents saves silently. Delete `design.modus.lock` if Device Configurator changes aren't reflected in generated code.
- **Memory Configurator ordering matters:** Always shrink regions first, then grow others. The tool validates intermediate states and will reject changes that temporarily exceed total memory.
- **I2C clock speed mismatch:** Waveshare 4.3" display requires ~184 kHz SCL (divider=31, highPhaseDutyCycle=16). Original project may have faster I2C settings for sensors — always compare against the `mtb-example-psoc-edge-lvgl-demo` reference settings.
- **Display init error `0x00AA2001`:** This is `CY_SCB_I2C_BAD_PARAM` from the display device-ID check, NOT an I2C bus failure. The display controller responded but returned wrong ID. Usually caused by: too-fast I2C clock, display MCU not booted, or another core driving the same SCB.
- **Do NOT reset display after GFXSS init:** Toggling `CYBSP_DISP_RST` after `Cy_GFXSS_Init()` breaks DSI sync. The official LVGL demo does not reset. Only reset if hardware corruption suspected.
- **PDL SCB re-init:** Call `Cy_SCB_I2C_Disable()` before any second `Cy_SCB_I2C_Init()` on the same SCB.
- **D-Cache coherency for framebuffers (CRITICAL):** Framebuffers in SOCMEM default to cacheable. MUST configure MPU non-cacheable region for `gfx_mem` BEFORE any framebuffer access. This is the #1 cause of garbled display output. MPU approach preferred over per-frame `SCB_CleanDCache()`.
- **D-Cache stale lines after cybsp_init() (CRITICAL):** Even with correct MPU non-cacheable config, stale D-Cache lines from the boot window (before MPU was configured) persist. MUST call `SCB_InvalidateDCache_by_Addr()` after `cybsp_init()` for IPC and framebuffer regions. See `mtb-multicore` agent Part 1.
- **MPU regions don't auto-sync (CRITICAL):** After changing Memory Configurator regions, MUST manually update MPU panel addresses in Device Configurator. Device Configurator warns about this but it's easily missed. Verify `cycfg_system.c` MPU regions match `cymem_CM55_0.h` defines.
- **Dead code on CM33 still compiles:** If CM33 had sensor code (e.g., `bme280_sensor.c`) that you disabled by removing the task but not the source file, it still compiles. Keep any `#define`s it references or remove the file entirely. `-Werror` makes unused-function warnings fatal.

### Quick-Reference Checklist: Adding LVGL Display to Existing PSOC Edge Project

1. ☐ Configure GFXSS personality in Device Configurator (do NOT replace entire `design.modus`)
2. ☐ Memory Configurator: add `gfx_mem` ≥ 2 MB, shrink first then grow, verify totals
3. ☐ MPU: add non-cacheable region for `gfx_mem` on CM55 + IPC shared memory on both cores
4. ☐ Save `design.modus` → verify `GeneratedSource/` files regenerated (check timestamps)
5. ☐ `common.mk`: `COMPONENTS+=GFXSS`, `CONFIG_DISPLAY=W4P3INCH_DISP` (not per-project Makefile)
6. ☐ `proj_cm55/Makefile`: `COMPONENTS+=FREERTOS RTOS_AWARE`, `DEFINES+=configENABLE_MVE=1`
7. ☐ CM55 `FreeRTOSConfig.h`: `configMINIMAL_STACK_SIZE=512`, `configENABLE_MVE=1`
8. ☐ Add `.mtb` deps to `proj_cm55/deps/` → `make getlibs`
9. ☐ Copy 10-15 hardware driver files from reference LVGL project (VG-Lite overrides, ports, `lv_conf.h`)
10. ☐ Create `retarget_io_init.c` on CM55 with PDL 3-step init pattern
11. ☐ CM55 `main.c`: `SCB_InvalidateDCache_by_Addr()` after `cybsp_init()` for IPC + gfx regions
12. ☐ Resolve SCB conflicts: migrate sensors off shared I2C bus, add bus recovery for SDA stuck LOW
13. ☐ Verify MPU addresses match Memory Configurator defines (`cycfg_system.c` vs `cymem_CM55_0.h`)
14. ☐ Delete `design.modus.lock` if Device Configurator changes don't take effect
15. ☐ `make clean && make -j8 build` all three cores — no undefined symbols, no new warnings

## 11. Top Pitfalls

1. **Missing GFXSS personality** → dozens of undefined symbols (§4)
2. **Framebuffers not in `.cy_gpu_buf`** → black screen or GPU hard fault
3. **Missing CY_IGNORE for LVGL originals** → duplicate symbol errors
4. **4.3" uses 832px width** (not 800) for GPU stride alignment
5. **`vg_lite_finish()` missing before FB swap** → screen tearing
6. **Stale DC notification not drained before buffer swap** → `ulTaskNotifyTake()` returns immediately with old VSYNC → buffer displayed before DC latches new address → persistent tearing on continuous animations. Fix: `ulTaskNotifyTake(pdTRUE, 0)` before `Cy_GFXSS_Set_FrameBuffer()` (see §6)
6. **FreeRTOS heap < 50 KB** → silent crash during `lv_init()`
7. **`lv_timer_handler()` returns 0** → clamp to 1ms or tasks starve
8. **FT5406 touch inverted** → invert x/y in read callback
9. **`COMPONENTS+=GFXSS` missing from `common.mk`** → `cy_graphics.h` not found
10. **D-cache stale lines after `cybsp_init()`** → IPC/shared-memory corruption
11. **Animating opacity on containers with children** → layer compositing every frame → flicker (§10)
12. **128-byte framebuffer alignment** → DC requires 128B-aligned base address and stride (not just 64B for GPU). Stride formula for RGB565: `ceil(width × 2 / 128) × 128`. Misalignment → `VG_LITE_NOT_ALIGNED` or display corruption
13. **Radial gradients unsupported** → `vg_lite_draw_radial_grad` returns `VG_LITE_NOT_SUPPORT` on GCNanoUltraV. Workaround: pre-render as image, use `vg_lite_draw_pattern`
14. **VGLite is single-thread only** → no concurrent contexts, no multi-threaded access. Must `vg_lite_close()` before `vg_lite_init()` in a different thread
15. **Arc paths require FP32** → `vg_lite_init_arc_path` silently fails with non-FP32 format
16. **`vg_lite_finish()` in hot path** → blocks CPU waiting for GPU. Use `vg_lite_flush()` (async) during frame rendering; `vg_lite_finish()` only at frame end
17. **Non-cacheable GPU memory** → framebuffers and VG-Lite heap should be in `.cy_gpu_buf` (non-cacheable MPU region). Alternative: cacheable + `SCB_CleanDCache()` after rendering
18. **Board rework for non-default displays** → 10.1" and 1.43" displays require resistor rework on KIT_PSE84_EVAL_EPC2 (§1 table)
19. **retarget-io not initialized on CM55** → `cy_retarget_io_mutex_initialized` assertion → `CY_HALT()`. Call `init_retarget_io()` even when CM33 uses same UART (§10b)
20. **I2C bus ownership conflict** → only one core may own a given SCB. Display/touch on SCB0 — disable or migrate CM33 I2C on same SCB (§10b)
21. **Display init `0x00AA2001`** → `CY_SCB_I2C_BAD_PARAM` from device-ID check. Fix I2C clock speed, verify display MCU booted, check no other core on same SCB
22. **Display reset after GFXSS init** → toggling `CYBSP_DISP_RST` after `Cy_GFXSS_Init()` breaks DSI sync. Do not reset post-init
23. **MPU region / Memory Configurator mismatch** → after changing memory regions, manually update MPU panel addresses. Verify `cycfg_system.c` MPU regions match `cymem_CM55_0.h` defines
24. **Device Configurator lock file** → `design.modus.lock` prevents saves silently. Delete if changes don't take effect
25. **`configMINIMAL_STACK_SIZE` too small** → graphics task needs 512 (32 KB). Non-graphics projects default to 128 → crash on `lv_init()` or `lv_timer_handler()`

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
- [ ] Framebuffer stride is 128-byte aligned (check: `width × bpp / 8` is multiple of 128)
- [ ] For 1.43" AMOLED: command mode transfer task created, `Cy_GFXSS_Transfer_Frame` called
- [ ] Board rework completed for non-default display (if applicable — see §1 table)
- [ ] `init_retarget_io()` called on CM55 (even when CM33 uses same UART)
- [ ] MPU non-cacheable region configured for `gfx_mem` (verify `cycfg_system.c` MPU addresses match `cymem_CM55_0.h`)
- [ ] `configMINIMAL_STACK_SIZE = 512` in CM55 `FreeRTOSConfig.h`
- [ ] No SCB0 ownership conflict between CM33 and CM55
- [ ] No `design.modus.lock` file blocking Device Configurator saves

## 12. LVGL Icon Font Pipeline (`lv_font_conv`)

Custom UI icons use `lv_font_conv` (Node.js tool) to generate LVGL-compatible C font files from TTF sources.

### Installation

```bash
npm install -g lv_font_conv    # or use npx lv_font_conv
lv_font_conv --version          # verify: 1.5.3+
```

### Font Sources

| Source | License | Use case |
|--------|---------|----------|
| `materialdesignicons-webfont.ttf` | SIL OFL | General UI icons (fire, snowflake, wifi, bluetooth, cog, etc.) |
| `MaterialSymbolsOutlined.ttf` | Apache 2.0 | Google Material Symbols (mode_heat_cool, etc.) |
| Custom `.ttf` files | Project | App-specific icons at Private Use Area codepoints |

### Generation Command

Generate at multiple sizes for different UI zones (status bar, buttons, hero icons):

```bash
# 20px — status bar icons
npx lv_font_conv --bpp 4 --size 20 \
  --font materialdesignicons-webfont.ttf \
    -r 0xF0026,0xF004D,0xF006A,0xF00AF,0xF0141,0xF01B4,0xF0210,0xF0238,0xF029A,0xF02DC,0xF02FC,0xF0425,0xF0432,0xF0493,0xF050F,0xF058C,0xF058E,0xF05A8,0xF05A9,0xF0717 \
  --font MaterialSymbolsOutlined.ttf -r 0xF16B \
  --format lvgl --no-compress --lv-include lvgl.h \
  -o proj_cm55/fonts/mdi_20.c

# Repeat with --size 28 and --size 36 for button and hero sizes
```

### Key Options

| Option | Value | Reason |
|--------|-------|--------|
| `--bpp` | `4` | 4-bit antialiasing — good quality/size tradeoff |
| `--no-compress` | — | Avoids runtime decompression overhead on MCU |
| `--format lvgl` | — | Generates `lv_font_t` compatible C source |
| `-r` | Hex codepoints | Comma-separated list; only include glyphs you use |
| Multiple `--font` | — | Merge glyphs from different TTF sources into one font |

### Integration

1. Place generated `.c` files in `proj_cm55/fonts/`
2. Declare fonts as `extern const lv_font_t mdi_20;` in a header (`mdi_icons.h`)
3. Define icon codepoints as UTF-8 string macros:
```c
#define MDI_FIRE     "\xF3\xB0\x84\x9D"  /* U+F0119 — fire/heat    */
#define MDI_SNOWFLK  "\xF3\xB0\x98\xAF"  /* U+F062F — snowflake    */
```
4. Use in LVGL: `lv_label_set_text(icon, MDI_FIRE);` with font set to `&mdi_28`

### Finding Codepoints

- **MDI Community:** Search at `pictogrammers.com/library/mdi/` → codepoint in URL
- **Google Material Symbols:** Download `MaterialSymbolsOutlined.codepoints` from Google Fonts GitHub → grep for icon name
- **Custom icons:** Assign to Private Use Area (U+E000–U+F8FF)

> **Tip:** The `lv_font_conv` command is preserved in the generated `.c` file header comment (line 4) for reproducibility. Always regenerate all sizes when changing the icon set.

## Related Code Examples

| CE Number | Name | Key Focus |
|---|---|---|
| CE238628 | Graphics using VGLite API | VGLite draw/blit APIs directly (no LVGL) |
| CE239259 | Graphics LVGL Demo | LVGL music player demo, FreeRTOS |
| CE239203 | Graphics using RLAD | RLAD compressed image display |
| CE239252 | CPU vs GPU Rendering | GPU-disabled vs GPU-enabled comparison |
| CE239252 | Single vs Double Buffering | Buffer mode comparison |
| CE239540 | DSI ULPM On Data Lane | MIPI DSI Ultra Low Power Mode |
| CE238541 | Smartwatch Demo using LVGL | 1.43" AMOLED + 4.3" TFT smartwatch GUI |

**GitHub:** `https://github.com/Infineon/mtb-example-psoc-edge-gfx-vglite`
