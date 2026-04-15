---
name: transparent-printf
description: Add transparent cross-core printf to a PSOC Edge E84 dual-core project. Routes CM33 printf output through a shared-memory ring buffer so both cores can use printf without UART conflicts. Uses --wrap=_write linker technique, IPC channel locks, and a drain task on the UART-owning core.
---

# Cross-Core Transparent Printf for PSOC Edge E84

> **Prerequisites:** Dual-core project with both CM33 and CM55 active. See /dual-core-setup for boot handshake. Project must be created with `project-creator-cli` — see /project-creation.

## Problem

On PSOC Edge E84, only one core can own the UART (typically CM55 owns SCB for retarget-io). The CM33 NS core has no direct UART access due to PPU (Protection Policy Unit) restrictions. Without this pattern, CM33 printf output is silently lost.

## Architecture

```
CM33 NS:  printf() → __wrap__write() → shared ring buffer (IPC locked)
CM55:     printf() → __wrap__write() → shared ring buffer (IPC locked)
CM55:     drain_task reads ring → UART output with [CM33]/[CM55] prefix
```

- **UART_OWNER (CM55):** Owns retarget-io, runs drain task
- **IPC_REMOTE (CM33 NS):** No retarget-io, writes only to ring buffer
- **Ring buffer:** MPSC (multi-producer, single-consumer) in shared SRAM
- **Lock:** Raw IPC channel lock (NOT PDL semaphore — see Pitfall #3)

---

## 1. Shared Ring Buffer Structure

Place in shared header (e.g., `shared/include/dual_printf_io.h`):

```c
#include "cy_pdl.h"

#define DUAL_PRINTF_RING_SIZE  (2048U)  /* Power of 2 recommended */
#define DUAL_PRINTF_BUF_SIZE   (256U)   /* Per-core line buffer */

typedef struct {
    volatile uint32_t head;             /* Next write position */
    volatile uint32_t tail;             /* Next read position */
    volatile uint8_t  ready;            /* Set after init */
    uint8_t           _pad[3];
    char              data[DUAL_PRINTF_RING_SIZE];
} dual_printf_ring_t;
```

### Storage — UART_OWNER only

```c
/* Only in CM55 build — CM33 gets address via IPC data register */
CY_SECTION_SHAREDMEM
static dual_printf_ring_t ring_storage __attribute__((aligned(32)));
```

> `CY_SECTION_SHAREDMEM` maps to non-cacheable shared SRAM — no explicit cache maintenance needed. Only DSB barriers after writes.

---

## 2. IPC Channel Lock (NOT PDL Semaphore)

Use raw IPC driver locks on a dedicated channel:

```c
#define IPC_MUTEX_CHANNEL  (9U)  /* Choose unused IPC channel */

static IPC_STRUCT_Type *ipc_mutex_base;

static inline void mutex_acquire(void)
{
    uint32_t timeout = 0;
    while (CY_IPC_DRV_SUCCESS != Cy_IPC_Drv_LockAcquire(ipc_mutex_base)) {
        if (++timeout > 100000UL) return;  /* Safety timeout */
    }
}

static inline void mutex_release(void)
{
    Cy_IPC_Drv_LockRelease(ipc_mutex_base, CY_IPC_NO_NOTIFICATION);
}
```

**Init:** `ipc_mutex_base = Cy_IPC_Drv_GetIpcBaseAddress(IPC_MUTEX_CHANNEL);`

---

## 3. Ring Buffer Write (both cores)

```c
static void ring_write(const char *data, size_t len)
{
    if (!ring_ptr || !ring_ptr->ready) return;

    mutex_acquire();
    __DSB();  /* Ensure previous writes visible */

    for (size_t i = 0; i < len; i++) {
        uint32_t next = (ring_ptr->head + 1) % DUAL_PRINTF_RING_SIZE;
        if (next == ring_ptr->tail) break;  /* Full — drop bytes */
        ring_ptr->data[ring_ptr->head] = data[i];
        ring_ptr->head = next;
    }

    __DSB();
    mutex_release();
}
```

---

## 4. `__wrap__write` — Linker Hook

This replaces the C library `_write()` function to intercept all printf output:

```c
int __wrap__write(int fd, const char *buf, int len)
{
    if (fd != 1 && fd != 2) return len;  /* Only stdout/stderr */

    /* Line-buffer, then flush complete lines to ring */
    for (int i = 0; i < len; i++) {
        line_buf[buf_pos++] = buf[i];
        if (buf[i] == '\n' || buf_pos >= DUAL_PRINTF_BUF_SIZE - 1) {
            line_buf[buf_pos] = '\0';
            ring_write(line_buf, buf_pos);
            buf_pos = 0;
        }
    }
    return len;
}
```

### Makefile (both `proj_cm33_ns/Makefile` and `proj_cm55/Makefile`)

```makefile
LDFLAGS+=-Wl,--wrap=_write
```

---

## 5. Drain Task (UART_OWNER / CM55 only)

```c
#define DRAIN_POLL_MS    (10U)
#define DRAIN_BATCH_SIZE (128U)

void dual_printf_drain_task(void *arg)
{
    (void)arg;
    char drain_buf[DRAIN_BATCH_SIZE];

    for (;;) {
        uint32_t count = 0;
        uint32_t tail = ring_ptr->tail;
        uint32_t head = ring_ptr->head;
        __DSB();

        while (tail != head && count < DRAIN_BATCH_SIZE) {
            drain_buf[count++] = ring_ptr->data[tail];
            tail = (tail + 1) % DUAL_PRINTF_RING_SIZE;
        }

        if (count > 0) {
            ring_ptr->tail = tail;
            __DSB();

            /* Write to UART via retarget-io (real _write, not wrapped) */
            extern int __real__write(int fd, const char *buf, int len);
            __real__write(1, drain_buf, (int)count);
        }

        vTaskDelay(pdMS_TO_TICKS(DRAIN_POLL_MS));
    }
}
```

**FreeRTOS task creation:**
```c
xTaskCreate(dual_printf_drain_task, "DRAIN",
            configMINIMAL_STACK_SIZE * 4, NULL,
            configMAX_PRIORITIES - 2, NULL);
```

Priority `MAX - 2`: High enough to drain promptly, below graphics and IPC tasks.

---

## 6. Initialization Sequence

### CM55 (UART_OWNER) — in main() or early task

```c
void dual_printf_init(void)
{
    /* Init ring buffer */
    memset(&ring_storage, 0, sizeof(ring_storage));
    ring_ptr = &ring_storage;
    ring_storage.ready = 1;
    __DSB();

    /* Init IPC mutex */
    ipc_mutex_base = Cy_IPC_Drv_GetIpcBaseAddress(IPC_MUTEX_CHANNEL);

    /* Publish ring address to CM33 via IPC data register */
    Cy_IPC_Drv_WriteDataValue(
        Cy_IPC_Drv_GetIpcBaseAddress(IPC_DATA_CHANNEL),
        (uint32_t)ring_ptr);

    initialized = true;
}
```

### CM33 NS (IPC_REMOTE) — after IPC init and CM55 boot

```c
void dual_printf_init(void)
{
    ipc_mutex_base = Cy_IPC_Drv_GetIpcBaseAddress(IPC_MUTEX_CHANNEL);

    /* Read ring address published by CM55 */
    uint32_t addr = 0;
    Cy_IPC_Drv_ReadDataValue(
        Cy_IPC_Drv_GetIpcBaseAddress(IPC_DATA_CHANNEL), &addr);
    ring_ptr = (dual_printf_ring_t *)addr;

    /* Wait for ring to be ready */
    uint32_t timeout = 0;
    while (!ring_ptr->ready && ++timeout < 1000000UL) {
        __NOP();
    }

    initialized = true;
}
```

---

## 7. Build Configuration

### Compile guards

Use `#define` to control which role each core plays:

```c
/* In proj_cm55/Makefile: */
DEFINES+=DUAL_PRINTF_ROLE_UART_OWNER

/* In proj_cm33_ns/Makefile: */
DEFINES+=DUAL_PRINTF_ROLE_IPC_REMOTE
```

### Shared source

Place `dual_printf_router.c` in a `shared/source/` directory and add to both Makefiles:

```makefile
SOURCES+=$(wildcard ../shared/source/*.c)
INCLUDES+=$(wildcard ../shared/include)
```

---

## 8. Common Pitfalls

1. **PDL semaphore breaks SRF IPC** — `Cy_IPC_Sema_Init()` overwrites the global `cy_semaIpcStruct` pointer, breaking the BSP's System Resource Framework. **Always use raw IPC driver locks** (`Cy_IPC_Drv_LockAcquire/Release`) on a dedicated channel.

2. **UART FIFO spin loop in drain** — Writing to UART with no timeout can hang the drain task when the FIFO is full. Add a retry limit (e.g., 200 iterations) per byte with `taskYIELD()` in the spin loop.

3. **D-cache stale reads** — If using cacheable SRAM (not `CY_SECTION_SHAREDMEM`), you must call `SCB_InvalidateDCache_by_Addr()` before reading ring buffer metadata on the consumer side.

4. **Ring buffer overflow** — If the producer writes faster than the drain reads, bytes are silently dropped. Increase `DUAL_PRINTF_RING_SIZE` or reduce printf volume. 2048 bytes handles typical debug output.

5. **Boot ordering** — CM55 must initialize the ring buffer and publish its address BEFORE CM33 calls `dual_printf_init()`. Use the boot handshake from /dual-core-setup to ensure correct ordering.

6. **IPC channel collision** — Verify your mutex channel and data channel don't conflict with IPC channels used by mtb-ipc, SRF, or other BSP subsystems. Channels 0–7 are typically reserved.

7. **`__real__write` not available** — The `__real__write` symbol only exists when `--wrap=_write` is in LDFLAGS. Without it, the drain task can't access the original UART write function.
