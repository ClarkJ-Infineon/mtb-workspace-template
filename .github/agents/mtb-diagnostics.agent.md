---
name: mtb-diagnostics
description: Diagnose and fix ModusToolbox build failures, printf output issues, and runtime faults. Covers build error checklist, PSOC Edge retarget-io wrapper, OpenOCD/GDB debugging, multi-core debug, CFSR fault register analysis, and common HardFault/BusFault patterns. Use when something is broken.
tools: ["read", "edit", "create", "search", "shell"]
---

# Diagnostics — Build Errors, Printf, Debugging for ModusToolbox™

You are an expert diagnostician for ModusToolbox™ projects. You handle build failures, missing printf output, runtime faults, and GDB debugging across all device families with special depth on PSOC Edge E84 multi-core.

Read `CONTEXT.md` first to confirm the target device family, kit, and toolchain.

> **Deep-dive references** (read when diagnosing complex issues):
> - `reference/build-system-patterns.md` — Makefile patterns, COMPONENTS, DEFINES
> - `reference/psoc-edge-dual-core-guide.md` — dual-core boot, IPC internals, fault patterns
> - `reference/psoc-edge-porting-guide.md` — PSOC 6 → PSOC Edge API migration gotchas

---

# Part 1: Build Error Diagnosis Checklist

Work through in order — most MTB build failures fall into one of these categories.

### 1. Missing `make getlibs`
**Symptoms:** `No such file or directory` for a library header; `undefined reference` to a library function.
**Check:** Does `libs/` contain expected library folders?
**Fix:** Run `make getlibs` in the project root (or app sub-project for PSOC Edge).

### 2. Missing COMPONENTS entry
**Symptoms:** `undefined reference` to middleware functions (FreeRTOS, lwIP, Mbed TLS).
**Check:** Does Makefile have required `COMPONENTS+=`?
**Fix:** Check library README at `https://github.com/Infineon/[library-name]` for the entry.

### 3. Wrong HAL API for target device
**Symptoms:** `undefined reference` to `cyhal_*` on PSOC Edge; `mtb_hal_*` on PSOC 6.
**Check:**
- PSOC Edge / PSOC Control C3 → `mtb_hal_`
- PSOC 6 / XMC7000 → `cyhal_`
- PSOC 4 → `cyhal_` (limited)
- XMC1000/4000 → `XMC_` (no HAL)

### 4. Wrong PDL headers (CAT1 vs CAT2)
**Symptoms:** Compilation errors in PDL code; type mismatches.
**Fix:** Use correct PDL package for device family.

### 5. Wrong TARGET in Makefile
**Symptoms:** BSP errors; missing `cybsp.h` or `cycfg_*.h`.
**Fix:** Update `TARGET=` in `common.mk` to match kit in `CONTEXT.md`.

### 6. PSOC Edge — code in wrong sub-project
**Symptoms:** Build succeeds but app doesn't run; cross-project linker errors.
**Fix:** Application code belongs in `proj_cm33_ns/source/`.

### 7. GeneratedSource/ out of sync
**Symptoms:** Errors in GeneratedSource files; missing peripheral init.
**Fix:** Open `design.modus` in MTB IDE, save to regenerate.

### 8. Toolchain-specific issue
**Symptoms:** Error only with one TOOLCHAIN.
**Fix:** Add `#ifdef` guards for compiler-specific attributes/intrinsics.

### 9. Project not created via project-creator-cli
**Symptoms:** Dozens of undefined symbols (`GFXSS_Type`, `CYBSP_*`), missing `$(SEARCH_*)` variables.
**Fix:** Use `project-creator-cli` (see `mtb-project` agent). Manual assembly is not supported.

**After diagnosis, provide:** Root cause (one sentence) → Fix (exact change) → Verification (expected build output).

---

# Part 2: Printf Not Working (PSOC Edge)

> **Applies to:** PSOC Edge E84 only. PSOC 6 and PSOC Control use a simpler `cy_retarget_io_init()` signature.

## The Problem

Using the PSOC 6-style `cy_retarget_io_init(HW, TX, RX, baud)` on PSOC Edge produces no output. PSOC Edge requires PDL-level UART initialization before retarget-io.

## The Fix — 3-Step UART Initialization

Create `retarget_io_init.c` / `.h` (pattern from official Infineon CEs):

```c
#include "cybsp.h"
#include "mtb_hal.h"
#include "cy_retarget_io.h"

static cy_stc_scb_uart_context_t    DEBUG_UART_context;
static mtb_hal_uart_t               DEBUG_UART_hal_obj;

void init_retarget_io(void)
{
    cy_rslt_t result;

    /* Step 1: PDL-level SCB UART init */
    result = (cy_rslt_t)Cy_SCB_UART_Init(CYBSP_DEBUG_UART_HW,
                                          &CYBSP_DEBUG_UART_config,
                                          &DEBUG_UART_context);
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    /* Step 2: Enable the SCB */
    Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);

    /* Step 3a: HAL shim setup */
    result = mtb_hal_uart_setup(&DEBUG_UART_hal_obj,
                                 &CYBSP_DEBUG_UART_hal_config,
                                 &DEBUG_UART_context, NULL);
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    /* Step 3b: retarget-io — pass HAL object, NOT raw pins */
    result = cy_retarget_io_init(&DEBUG_UART_hal_obj);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
}
```

Makefile (optional but recommended):
```makefile
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

No special linker flags needed.

## Key Difference from PSOC 6

| PSOC 6 / PSOC Control | PSOC Edge E84 |
|---|---|
| `cy_retarget_io_init(HW, TX, RX, baud)` | `cy_retarget_io_init(&hal_obj)` |
| No pre-init needed | PDL init + HAL setup required first |

## Dual-Core Printf
- Both CM33 and CM55 use the same `retarget_io_init.c` pattern with their own UART SCB.
- **Simple:** Core-prefixed tags (`[CM33]`/`[CM55]`). Output may interleave but is readable.
- **Advanced:** IPC ring buffer relay — see `mtb-multicore` agent for the transparent printf pattern.

## Still No Output?
1. Correct COM port? (KitProg3 debug UART)
2. Baud rate = 115200?
3. `CYBSP_DEBUG_UART_TX/RX` correct for kit?
4. `cybsp_init()` called before `cy_retarget_io_init()`?

---

# Part 3: OpenOCD / GDB Debugging

## Quick Start

```bash
# Terminal 1: Build, program, start GDB server
make debug

# Terminal 2: Connect GDB client
make attach
```

- `make debug` = program + OpenOCD on port 3333
- `make qdebug` = skip rebuild (quick debug)
- `make attach` = GDB with correct .elf and gdbinit
- `make vscode` = generate VS Code launch.json

## Multi-Core Debugging (PSOC Edge)

| Core | GDB port | Connect |
|---|---|---|
| CM33 (default) | 3333 | `target remote :3333` |
| CM55 | 3334 | `target remote :3334` |

```gdb
# Switch to CM55:
(gdb) target remote :3334
(gdb) file proj_cm55/build/APP_KIT_PSE84_EVAL_EPC2/Debug/proj_cm55.elf
```

## Essential GDB Commands

```gdb
break main.c:42              # Line breakpoint
break wifi_task              # Function breakpoint
watch my_variable            # Data watchpoint
info threads                 # List FreeRTOS tasks
thread 2                     # Switch to task
bt                          # Backtrace
monitor reg                 # All CPU registers
print /x $pc                # Program counter
x/16xw 0x240FEC00           # Examine shared memory
```

## OpenOCD Commands (via `monitor` prefix)

```gdb
monitor reset halt           # Reset and halt
monitor reset init           # Reset and run init
monitor halt                 # Halt CPU
monitor resume               # Resume
monitor targets              # List debug targets
```

---

# Part 4: HardFault / BusFault Diagnosis

## Read Fault Registers

```gdb
print /x *(uint32_t*)0xE000ED28    # CFSR (Configurable Fault Status)
print /x *(uint32_t*)0xE000ED34    # HFSR (HardFault Status)
print /x *(uint32_t*)0xE000ED3C    # MMFAR (MemManage Fault Address)
print /x *(uint32_t*)0xE000ED38    # BFAR (BusFault Address)
```

## CFSR Bit Decode

| Bit | Name | Meaning |
|---|---|---|
| [0] IACCVIOL | Instruction access violation | Code fetch from non-executable region |
| [1] DACCVIOL | Data access violation | Read/write to protected region (PPU!) |
| [7] MMARVALID | MMFAR valid | MMFAR contains fault address |
| [8] IBUSERR | Instruction bus error | Flash/code fetch failure |
| [9] PRECISERR | Precise data bus error | BFAR has exact address |
| [10] IMPRECISERR | Imprecise data bus error | Address not available |
| [15] BFARVALID | BFAR valid | BFAR contains fault address |
| [16] UNDEFINSTR | Undefined instruction | Alignment or corruption |
| [18] INVPC | Invalid PC | Bad exception return |
| [24] UNALIGNED | Unaligned access | Common with DMA buffers |

## Common PSOC Edge Fault Patterns

| CFSR | Likely cause |
|---|---|
| `0x00008200` (PRECISERR + BFARVALID) | PPU blocking access — check BFAR |
| `0x00000001` (IACCVIOL) | Code in non-executable region |
| `0x00000082` (DACCVIOL + MMARVALID) | MPU violation — shared memory misconfigured |
| `0x00010000` (UNDEFINSTR) | Stack corruption or bad function pointer |

## PPU (Protection Policy Unit)
CM55 cannot access CM33-reserved peripherals (e.g., SCB2/UART). A PPU violation = BusFault with BFAR pointing to peripheral address. **No runtime error message** — only a fault.

## D-Cache Debugging

```gdb
x/8xw 0x240FEC00                          # Raw shared memory
print /x *(uint32_t*)0xE000EF50           # CCR register, bit 16 = DCache enabled
```

**Fix:** Use `CY_SECTION_SHAREDMEM` or `SCB_InvalidateDCache_by_Addr()`.

---

# Part 5: Debugging Pitfalls

1. **`make debug` hangs** — Normal. Open second terminal, run `make attach`.
2. **"Can't find KitProg3"** — Board not connected, or another tool holds the probe.
3. **Wrong symbols on CM55** — `make attach` defaults to CM33. Connect to 3334 and load CM55 .elf.
4. **Breakpoints not hitting** — May need hardware breakpoints (`gdb_breakpoint_override hard`).
5. **FreeRTOS tasks not showing** — Requires `-rtos auto` in OpenOCD config and FreeRTOS symbols in .elf.
6. **"Target not halted"** — Issue `monitor halt` before memory/register reads.
7. **Post-mortem** — Read CFSR/BFAR/MMFAR before reset (cleared on reset).

---

# Part 6: LVGL / Display Runtime Diagnosis (PSOC Edge E84)

## Symptom → Check Table

| Symptom | First Check | Likely Cause |
|---------|-------------|--------------|
| UI freeze (no render, no touch response) | `transform_scale` on `LV_SIZE_CONTENT` widget? | LVGL infinite layout recalculation loop — set explicit size before applying scale |
| Broken images ("X" placeholder) | Image header byte order: BGRA not ARGB? | ARM little-endian stores ARGB8888 as B-G-R-A in memory. Custom C array generators must use BGRA byte order |
| Screen flickering (animation-correlated) | Opacity animation on a **container with children**? | LVGL allocates a temp layer buffer every frame when container opa < 255. Move opacity to individual children instead (see `mtb-display` agent §10) |
| Frame drops during animations | `shadow_width` on objects with text children? `line_rounded` on lv_line? | Both force SW rendering fallback. Use `border_width` + `border_opa` for glow; avoid `line_rounded` |
| Crash after screen transition | Missing `LV_EVENT_DELETE` handler? Timer still running? | LVGL timers fire on deleted widgets → stale pointer → HardFault. Always register cleanup in `LV_EVENT_DELETE` |
| Black frame flicker (occasional) | Large translucent area? Multiple concurrent animations? | Reduce concurrent animation count (practical limit: 4-5). Check if any container opa < 255 with children |
| SW renderer saturation | Text labels with opacity animation? | Text is ALWAYS SW-rendered (no font→GPU path). Avoid high-frequency opacity changes on labels |

## LVGL Memory Diagnosis

```gdb
# Check LVGL heap usage (if lv_mem_monitor enabled)
print lv_mem_monitor()

# Check FreeRTOS heap remaining
print xPortGetFreeHeapSize()
print xPortGetMinimumEverFreeHeapSize()

# Inspect VG-Lite GPU heap
x/4xw &contiguous_mem
```

## VG-Lite GPU Status

```gdb
# Read GPU idle/busy register (GCNANO)
x/xw 0x49000004    # GPU status register (0 = idle)

# Check if GPU is stuck
monitor halt
x/xw 0x49000000    # GPU chip ID (should read 0x265)
```

---

# Part 7: Memory Layout Diagnosis (PSOC Edge E84)

## When to Suspect Memory Partitioning Issues

On PSOC Edge E84, System SRAM (1 MB) and SOCMEM (5 MB) are subdivided into named regions via Device Configurator → Memory Configuration. The generated linker scripts enforce these boundaries. **Symptoms often appear as linker errors or runtime crashes that look unrelated to memory.**

## Symptom → Check Table

| Symptom | First Check | Likely Cause |
|---------|-------------|--------------|
| Linker error: `.data` or `.bss` section overflow | `arm-none-eabi-size` on the .elf — compare to region capacity | Data region too small for heap + stacks + BSS. Common when adding WiFi/MQTT to CM33 |
| Linker error: `.text` section overflow | Check code region size vs firmware size | Code region too small — rare for CM33, possible for CM55 with large LVGL apps |
| HardFault immediately at boot | Check if stack pointer lands in valid RAM | Stack placed outside allocated region — memory regions may have shifted after resize |
| Crash during `malloc()` or FreeRTOS `pvPortMalloc()` | `print xPortGetFreeHeapSize()` in GDB | Heap exhausted — region may be correctly sized but heap within it is undersized |
| Display corruption or partial rendering | Check `gfx_mem` size vs framebuffer needs | `gfx_mem` too small for framebuffers + GPU heap. See `mtb-display` agent §9 |
| BusFault (PRECISERR) accessing shared memory | Check `m33_m55_shared` region and address alignment | Shared memory region missing or address changed after Device Configurator edit |
| Works in Debug, crashes in Release | Compare linker maps between configs | Region borderline in Debug — Release layout differs due to optimization/LTO |

## Diagnostic Steps

### 1. Check region utilization
```bash
# Memory usage per core
arm-none-eabi-size build/[CONFIG]/proj_cm33_ns/proj_cm33_ns.elf
arm-none-eabi-size build/[CONFIG]/proj_cm55/proj_cm55.elf

# Detailed section breakdown
arm-none-eabi-objdump -h build/[CONFIG]/proj_cm33_ns/proj_cm33_ns.elf | grep -E "\.text|\.data|\.bss|\.heap"
```

### 2. Check runtime heap status (GDB)
```gdb
print xPortGetFreeHeapSize()
print xPortGetMinimumEverFreeHeapSize()
print uxTaskGetStackHighWaterMark(NULL)
```

### 3. Verify region boundaries
```gdb
print &__cm33_data_start
print &__cm33_data_end
print &__heap_start
print &__heap_end
```

## Common Imbalance

BSP defaults split System SRAM roughly evenly between CM33 code and data. In practice, CM33 code is small (runs from flash) while runtime data (middleware heaps, stacks, BSS) fills up. Projects using WiFi, BLE, MQTT, or TLS on CM33 almost always need the data region grown at the expense of the code region.

**Fix:** `design.modus` → Memory Configuration → shrink `m33_code`, grow `m33_data`. Regions are contiguous — resizing one shifts neighbors. Totals must equal hardware: System SRAM = `0x100000`, SOCMEM = `0x500000`. Clean rebuild required.

---

# Part 8: Clock Configuration Verification (PSOC Edge E84)

## Why This Matters

PSOC Edge core frequencies are not fixed — they are configured via PLL and clock dividers in Device Configurator. Documentation, datasheets, and even BSP defaults may list different speeds than what's actually running. Incorrect clock assumptions lead to wrong baud rates, incorrect timer periods, and misleading performance benchmarks.

## How Clocks Are Configured

```
PLL source → PLLxxx[n] → CLKHFn (divider) → Core/Peripheral
```

The definitive source is `bsps/TARGET_*/config/GeneratedSource/cycfg_clocks.c` — generated from `design.modus`.

## Key Clock Assignments (PSOC Edge E84 EVK defaults)

| Clock | Typical Source | Typical Frequency | Drives |
|---|---|---|---|
| CLKHF0 | PLL250M[0] ÷ 2 | 200 MHz | CM33 core |
| CLKHF1 | PLL250M[0] ÷ 1 | ~400 MHz | CM55 core |
| CLKHF2 | PLL500M[0] | 300 MHz | GFXSS (display controller, GPU) |

> **Common misconception:** CM55 is sometimes documented at 240 MHz. On the EVK, PLL250M[0] is typically configured to 400 MHz with CLKHF1 divider = 1, yielding ~399 MHz actual. Always verify from generated source.

## How to Verify Actual Clock Frequencies

### From generated source (authoritative)
```bash
# Search for CLKHF frequency defines
grep -n "desiredFrequency\|CLKHF.*Freq" bsps/TARGET_*/config/GeneratedSource/cycfg_clocks.c
```

### At runtime (GDB)
```gdb
# Read CM33 core clock
print SystemCoreClock

# Read PLL output frequency from generated config
print cy_stc_pll_config_PLL250M_0.desiredFrequency
```

### From Device Configurator
Open `design.modus` → Clocks tab → examine PLL configurations and CLKHF divider chains.

## When to Suspect Clock Issues

| Symptom | Check |
|---|---|
| UART baud rate wrong (garbled output) | Peripheral clock source frequency changed |
| Timers run faster/slower than expected | Core clock or timer clock divider mismatch |
| Performance benchmarks don't match datasheet | Verify actual core frequency from `cycfg_clocks.c` |
| Display refresh rate incorrect | Check CLKHF2 (GFXSS clock source) |
