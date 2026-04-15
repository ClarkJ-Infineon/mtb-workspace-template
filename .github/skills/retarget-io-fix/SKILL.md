---
name: retarget-io-fix
description: Fix printf not producing output on PSOC Edge E84 devices. Use when serial output is missing, printf produces no output, retarget-io initialization fails, or CM55 needs printf access on PSOC Edge.
---

# Retarget-IO Fix — PSOC Edge Printf Wrapper

Fix for `printf()` not producing output on PSOC Edge E84 devices.

> **Applies to:** PSOC Edge E84 only. PSOC 6 and other families work with standard `cy_retarget_io_init()`.

---

## The Problem

On PSOC Edge, calling `cy_retarget_io_init(CYBSP_DEBUG_UART_HW, CYBSP_DEBUG_UART_TX, CYBSP_DEBUG_UART_RX, 115200)` alone does **not** enable printf output. The function returns `CY_RSLT_SUCCESS` but no characters appear on the terminal.

This is because PSOC Edge uses a different UART abstraction layer than PSOC 6. The retarget-io library needs additional initialization to route `_write()` syscalls through the correct UART driver.

---

## The Fix

Use the `--wrap=_write` linker flag and a custom `_write` wrapper:

### Step 1: Add linker flag to Makefile

In your app Makefile (`proj_cm33_ns/Makefile` for PSOC Edge):

```makefile
LDFLAGS+=-Wl,--wrap=_write
```

### Step 2: Create the wrapper function

Add to your `main.c` or a dedicated `retarget_io_wrapper.c`:

```c
#include "cy_retarget_io.h"
#include <unistd.h>

/* Wrap _write to route through retarget-io UART */
int __wrap__write(int fd, const char *ptr, int len)
{
    (void)fd;  /* Ignore file descriptor — all output goes to debug UART */

    for (int i = 0; i < len; i++)
    {
        cy_retarget_io_putchar(ptr[i]);
    }
    return len;
}
```

### Step 3: Initialize retarget-io normally

```c
#include "cy_retarget_io.h"

void init_debug_uart(void)
{
    cy_rslt_t result = cy_retarget_io_init(
        CYBSP_DEBUG_UART_HW,
        CYBSP_DEBUG_UART_TX,
        CYBSP_DEBUG_UART_RX,
        115200);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
}
```

Call `init_debug_uart()` **after** `cybsp_init()` in `main()`.

---

## Dual-Core Printf

If both CM33 and CM55 print simultaneously, output interleaves at the character level (both share one physical UART). Two approaches:

### Option A: Core-prefixed output (simple, recommended)

Each core prefixes its printf with a tag. Output still interleaves but is readable:

```c
/* CM33 side */
#define LOGM(fmt, ...) printf("[CM33] " fmt "\n", ##__VA_ARGS__)

/* CM55 side */
#define LOGC(fmt, ...) printf("[CM55] " fmt "\n", ##__VA_ARGS__)
```

### Option B: IPC-routed logging (advanced)

CM55 sends log messages via IPC to CM33, which serializes all output through a single printf task. Higher complexity, but output never interleaves. See the /ipc-patterns skill for the message queue pattern.

> **Important:** CM55 cannot directly access the UART (SCB2) on PSOC Edge — the Protection Policy Unit (PPU) blocks it. CM55 printf **must** use an IPC relay to CM33. This is a hardware-enforced restriction, not a software bug.

---

## Verification

After applying the fix, you should see output in your terminal (115200 baud, 8N1):

```
[CM33] cybsp_init complete
[CM33] retarget-io initialized
[CM33] WiFi connecting...
```

If still no output:
1. Verify the correct COM port is selected (KitProg3 debug UART)
2. Check baud rate matches (115200)
3. Verify `CYBSP_DEBUG_UART_TX/RX` pins are correct for your kit
4. Ensure `cybsp_init()` is called before `cy_retarget_io_init()`
