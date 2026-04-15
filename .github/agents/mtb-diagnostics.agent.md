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

> **Applies to:** PSOC Edge E84 only. PSOC 6 works with standard `cy_retarget_io_init()`.

## The Problem

`cy_retarget_io_init()` returns success but no output appears. PSOC Edge needs a `_write` wrapper.

## The Fix

### Step 1: Linker flag
```makefile
# proj_cm33_ns/Makefile
LDFLAGS+=-Wl,--wrap=_write
```

### Step 2: Wrapper function
```c
#include "cy_retarget_io.h"
#include <unistd.h>

int __wrap__write(int fd, const char *ptr, int len)
{
    (void)fd;
    for (int i = 0; i < len; i++)
        cy_retarget_io_putchar(ptr[i]);
    return len;
}
```

### Step 3: Init after cybsp_init()
```c
cy_rslt_t result = cy_retarget_io_init(
    CYBSP_DEBUG_UART_HW, CYBSP_DEBUG_UART_TX, CYBSP_DEBUG_UART_RX, 115200);
CY_ASSERT(result == CY_RSLT_SUCCESS);
```

## Dual-Core Printf
- **Simple:** Core-prefixed macros (`[CM33]`/`[CM55]` tags). Output interleaves but is readable.
- **Advanced:** IPC ring buffer relay — see `mtb-multicore` agent for the transparent printf pattern.
- **Important:** CM55 cannot access UART (PPU blocks SCB2). CM55 printf **must** use IPC relay.

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
