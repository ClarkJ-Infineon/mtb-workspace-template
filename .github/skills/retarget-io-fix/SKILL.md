---
name: retarget-io-fix
description: Fix printf not producing output on PSOC Edge E84 devices. Covers the correct PDL→HAL→retarget-io initialization sequence for CM33 and CM55 cores. Use when serial output is missing or retarget-io initialization fails on PSOC Edge.
---

# Retarget-IO on PSOC Edge E84 — Correct Initialization

How to get `printf()` working on PSOC Edge E84 devices (CM33 and CM55 cores).

> **Applies to:** PSOC Edge E84 only. PSOC 6 and PSOC Control use a simpler `cy_retarget_io_init()` signature that takes HW/TX/RX/baud directly.

---

## The Problem

On PSOC Edge, calling the PSOC 6-style `cy_retarget_io_init(HW, TX, RX, baud)` does **not** produce printf output. The function may compile (or appear to succeed) but no characters appear on the terminal.

PSOC Edge uses a different UART abstraction layer. The retarget-io library requires a fully initialized `mtb_hal_uart_t` HAL object — not raw pin/HW parameters. You must perform PDL-level SCB UART initialization first, then pass the HAL object to `cy_retarget_io_init()`.

---

## The Fix — 3-Step UART Initialization

This pattern is used by all official Infineon PSOC Edge code examples. Create two files: `retarget_io_init.c` and `retarget_io_init.h`.

### retarget_io_init.h

```c
#ifndef _RETARGET_IO_INIT_H_
#define _RETARGET_IO_INIT_H_

#include "cybsp.h"
#include "mtb_hal.h"
#include "cy_retarget_io.h"

/* Macros for deepsleep callback (set RTS to NULL if unused) */
#define DEBUG_UART_RTS_PORT     (NULL)
#define DEBUG_UART_RTS_PIN      (0U)

void init_retarget_io(void);

#endif /* _RETARGET_IO_INIT_H_ */
```

### retarget_io_init.c

```c
#include "retarget_io_init.h"

/* UART context and HAL objects */
static cy_stc_scb_uart_context_t    DEBUG_UART_context;
static mtb_hal_uart_t               DEBUG_UART_hal_obj;

void init_retarget_io(void)
{
    cy_rslt_t result;

    /* Step 1: Initialize the SCB UART at PDL level */
    result = (cy_rslt_t)Cy_SCB_UART_Init(CYBSP_DEBUG_UART_HW,
                                          &CYBSP_DEBUG_UART_config,
                                          &DEBUG_UART_context);
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    /* Step 2: Enable the SCB block */
    Cy_SCB_UART_Enable(CYBSP_DEBUG_UART_HW);

    /* Step 3a: Set up the HAL shim over the PDL driver */
    result = mtb_hal_uart_setup(&DEBUG_UART_hal_obj,
                                 &CYBSP_DEBUG_UART_hal_config,
                                 &DEBUG_UART_context, NULL);
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    /* Step 3b: Initialize retarget-io — pass the HAL object, NOT raw pins */
    result = cy_retarget_io_init(&DEBUG_UART_hal_obj);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
}
```

### Makefile

Add the LF→CRLF conversion define (optional but recommended for terminal readability):

```makefile
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF
```

No special linker flags are needed.

### Usage in main.c

```c
#include "retarget_io_init.h"

int main(void)
{
    cybsp_init();
    init_retarget_io();

    printf("Hello from PSOC Edge!\r\n");
    /* ... */
}
```

Call `init_retarget_io()` **after** `cybsp_init()`.

---

## Key Differences from PSOC 6 / PSOC Control

| | PSOC 6 / PSOC Control | PSOC Edge E84 |
|---|---|---|
| **API signature** | `cy_retarget_io_init(HW, TX, RX, baud)` | `cy_retarget_io_init(&hal_obj)` |
| **Pre-init required** | None — retarget-io handles everything | Yes — PDL init + HAL setup first |
| **Config source** | Baud rate passed as parameter | UART config from Device Configurator (`CYBSP_DEBUG_UART_config`) |
| **Typical files** | Just `#include "cy_retarget_io.h"` in main.c | Separate `retarget_io_init.c` / `.h` (official pattern) |

---

## CM55 Printf

CM55 on PSOC Edge uses the **same pattern** as CM33. Each core project (`proj_cm33_ns/` and `proj_cm55/`) gets its own `retarget_io_init.c` with the identical 3-step sequence, pointing to its own configured UART SCB.

> **CRITICAL:** CM55 projects MUST call `init_retarget_io()` even if CM33 has already initialized the same UART (SCB2). The retarget-io library uses a per-core FreeRTOS mutex. If CM55 calls `printf()` without having initialized retarget-io, the mutex is uninitialized → FreeRTOS assertion failure → `CY_HALT()` crash. Both cores sharing SCB2 is safe — output interleaves at byte level, which is normal and expected.

> **Note:** If both cores print to the same physical UART simultaneously, output may interleave at the character level. Prefix output with core tags for readability:
>
> ```c
> /* CM33 side */
> printf("[CM33] Starting WiFi...\r\n");
>
> /* CM55 side */
> printf("[CM55] ML inference running\r\n");
> ```
>
> For guaranteed non-interleaved output, route CM55 log messages through IPC to CM33 and serialize through a single print task. See the /ipc-patterns skill for message queue patterns.

---

## Troubleshooting

If printf still produces no output after applying the correct initialization:

1. **Verify Device Configurator** — `CYBSP_DEBUG_UART` must be configured as a UART personality in `design.modus`. The PDL config structs (`CYBSP_DEBUG_UART_config`, `CYBSP_DEBUG_UART_hal_config`) are generated from this.
2. **Check COM port** — use the KitProg3 debug UART port (not the general USB serial)
3. **Check baud rate** — must match the Device Configurator UART settings (typically 115200)
4. **Call order** — `cybsp_init()` must complete before `init_retarget_io()`
5. **Correct core project** — ensure each core's Makefile includes its own source files

---

## Matter SDK Logging Impact

On Matter projects, the CHIP SDK's `CHIP_DETAIL_LOGGING` generates thousands of log lines during commissioning. This floods the shared UART and can stall CM55's LVGL frame updates.

**Fix:** Edit `mtb_shared/connectedhomeip/src/core/CHIPBuildConfig.h` directly and set `CHIP_DETAIL_LOGGING` to 0. Project Makefile `DEFINES` do NOT override this — the SDK header is included after project config and unconditionally redefines the macro. You must take library ownership (`make modlibs`) to preserve the edit across `make getlibs`.

---

## Reference

Pattern sourced from official Infineon PSOC Edge code examples:
- `Infineon/mtb-example-psoc-edge-hello-world` (CM33)
- `Infineon/mtb-example-psoc-edge-ml-deepcraft-deploy-motion` (CM55)
