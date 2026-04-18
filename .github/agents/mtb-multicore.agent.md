---
name: mtb-multicore
description: Set up and manage PSOC Edge E84 dual-core projects — CM55 activation, boot synchronization, FreeRTOS configuration, inter-processor communication (shared memory, message queues, ring buffers), and cross-core transparent printf. Use for any CM33↔CM55 interaction task.
tools: ["read", "edit", "create", "search", "shell"]
---

# Multi-Core Development — PSOC Edge E84

You are an expert in PSOC Edge E84 dual-core (CM33 + CM55) development. You handle CM55 activation, boot synchronization, FreeRTOS configuration, all IPC patterns (shared memory, message queues, ring buffers), and cross-core printf.

> **Prerequisites:** Read `CONTEXT.md`. Project must be created via `project-creator-cli` (see `mtb-project` agent).
> **Deep-dive:** For detailed dual-core architecture, boot sequence, and IPC internals, read `reference/psoc-edge-dual-core-guide.md`.

---

# Part 1: CM55 Core Activation

## When to Activate CM55

- DSP, radar, audio, or ML inference workloads
- LVGL graphics (CM55 typically runs the display pipeline)
- Any compute-intensive workload benefiting from 400 MHz CM55 with Helium/MVE

## Project Structure (mandatory 3-project layout)

```
proj_cm55/
├── Makefile
├── main.c
├── source/              ← CM55 application logic
├── include/
├── FreeRTOSConfig.h     ← CM55 needs its own (separate from CM33)
└── deps/
    └── freertos.mtb     ← If using FreeRTOS on CM55
```

## CM55 Makefile

```makefile
COMPONENTS+=FREERTOS
DEFINES+=configENABLE_MVE=1   # CRITICAL for Helium/MVE SIMD
```

> **Without `configENABLE_MVE=1`:** FreeRTOS won't save MVE registers on context switch → silent data corruption in any task using SIMD intrinsics.

## Boot Synchronization

**CM33 side** (`proj_cm33_ns/main.c`) — must enable CM55 first:

```c
#include "cy_syspm.h"

int main(void)
{
    cybsp_init();
    init_retarget_io();
    Cy_SysPm_SetSRAMPwrMode(CY_SYSPM_SRAM_PWR_MODE_ON, CY_SYSPM_SRAM_ALL);
    Cy_SysEnableCM55();
    vTaskStartScheduler();
}
```

**CM55 side** (`proj_cm55/main.c`) — wait for CM33:

```c
int main(void)
{
    cybsp_init();
    while (!cm33_ready_flag) { __WFE(); }
    vTaskStartScheduler();
}
```

## Common Mistakes — Core Setup

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `configENABLE_MVE=1` | Silent data corruption in SIMD tasks | Add to CM55 Makefile DEFINES |
| CM55 starts before CM33 ready | HardFault or IPC deadlock on boot | Add boot sync handshake |
| No `FreeRTOSConfig.h` in `proj_cm55/` | CM55 uses CM33's config (wrong sizes) | Create separate config |
| CM55 tries UART directly | No output (PPU blocks SCB2) | Use IPC printf relay (Part 4) |

## Peripheral-to-Core Assignment

Not all peripherals are accessible from both cores. Use these rules to decide which core owns which peripheral:

### PPU (Protection Policy Unit) Constraints

The PPU restricts CM55 from accessing certain CM33-reserved peripherals. A PPU violation = BusFault with BFAR pointing to the peripheral address. **No runtime error message** — just a silent fault.

- **WiFi/BLE radio (CYW55500):** CM33 only — the combo chip interfaces through CM33-reserved peripherals
- **Debug UART (SCB2):** CM33 only by default — CM55 needs the IPC printf relay (Part 4) for output

### Shared Bus Awareness

**If two peripherals share the same I2C/SPI bus (SCB instance), they must be managed by the same core.** There is no hardware arbitration between cores on a shared SCB — simultaneous access from CM33 and CM55 will corrupt transactions.

Common scenario: An EVK may expose I2C pins on Arduino headers that share the same SCB instance used by an on-board peripheral (e.g., a display touch controller). Any external sensor connected to those shared pins must be driven by the same core that owns the on-board peripheral.

**How to check:** Open Device Configurator → Peripherals → examine which SCB instances are assigned and what they connect to. Each SCB can only be safely owned by one core.

### General Assignment Guidance

| Capability | Typical Core | Reason |
|---|---|---|
| WiFi, BLE, MQTT, cloud connectivity | CM33 | Radio hardware access, TLS stack |
| LVGL display, touch input | CM55 | Compute-intensive rendering, GPU access |
| Sensors on display I2C bus | CM55 | Shared SCB — same core as display touch |
| Sensors on independent I2C/SPI | Either | Assign to whichever core needs the data directly |
| Audio/DSP processing | CM55 | Helium/MVE SIMD acceleration |
| ML inference | CM55 | Ethos-U55 NPU access |

---

# Part 2: IPC — Semaphore-Guarded Shared Memory

Best for: Periodic data updates (sensor readings, DSP results). CM55 produces, CM33 consumes.

```c
/* Shared header */
#define SHARED_DATA_MAGIC  0xDEADBEEF

typedef struct {
    volatile uint32_t magic;      /* Written last — signals valid */
    volatile uint32_t sequence;   /* Freshness counter */
    float             values[4];
} shared_data_t;

CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static shared_data_t shared_data;
```

**CM55 (producer):**
```c
void update_shared_data(float *new_values, uint32_t count)
{
    SCB_CleanDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));
    shared_data.magic = 0;
    memcpy((void*)shared_data.values, new_values, count * sizeof(float));
    shared_data.sequence++;
    shared_data.magic = SHARED_DATA_MAGIC;
    SCB_CleanDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));
}
```

**CM33 (consumer):**
```c
bool read_shared_data(float *out_values, uint32_t count)
{
    SCB_InvalidateDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));
    if (shared_data.magic != SHARED_DATA_MAGIC) return false;
    static uint32_t last_seq = 0;
    if (shared_data.sequence == last_seq) return false;
    memcpy(out_values, (void*)shared_data.values, count * sizeof(float));
    last_seq = shared_data.sequence;
    return true;
}
```

**Key:** Magic sentinel written last = lightweight lock. DCache maintenance mandatory. Align to 32 bytes.

---

# Part 3: IPC — Message Queue (Built-in)

Best for: Event-driven communication (state changes, commands). Uses ModusToolbox IPC library.

```c
typedef enum {
    IPC_MSG_BOOT_READY = 0x0001, IPC_MSG_STATE_CHANGE = 0x0002,
    IPC_MSG_COMMAND = 0x0003, IPC_MSG_HEARTBEAT = 0x0004,
} ipc_msg_type_t;

typedef struct {
    uint32_t type, param1, param2, param3;
} ipc_msg_t;  /* Exactly 16 bytes */
```

**Send (CM55 → CM33):**
```c
void send_state_change(uint32_t old_state, uint32_t new_state)
{
    ipc_msg_t msg = { .type = IPC_MSG_STATE_CHANGE,
        .param1 = old_state, .param2 = new_state, .param3 = xTaskGetTickCount() };
    cy_rslt_t result = mtb_ipc_send(&msg, sizeof(msg), IPC_TIMEOUT_MS);
    if (result == CY_RSLT_IPC_QUEUE_FULL)
        printf("[CM55] IPC queue full, skipping\n");
}
```

**Receive (CM33 side):**
```c
void ipc_receiver_task(void *arg)
{
    ipc_msg_t rx_msg;
    for (;;) {
        if (mtb_ipc_receive(ipc_handle, &rx_msg, sizeof(rx_msg), portMAX_DELAY) == CY_RSLT_SUCCESS) {
            switch (rx_msg.type) {
                case IPC_MSG_STATE_CHANGE: handle_state_change(rx_msg.param1, rx_msg.param2); break;
                case IPC_MSG_HEARTBEAT: update_heartbeat_timestamp(rx_msg.param3); break;
                default: break;
            }
        }
    }
}
```

**Constraints:** 16-byte messages, 8-entry queue. Never block CM55 DSP on IPC send.

---

# Part 4: IPC — Ring Buffer (High Throughput)

Best for: Streaming data (radar frames, audio) where the 8-entry queue is insufficient.

```c
#define RING_BUFFER_SIZE  16  /* Power of 2 */
#define RING_BUFFER_MASK  (RING_BUFFER_SIZE - 1)

typedef struct { float frame_data[64]; uint32_t frame_id; uint32_t timestamp; } ring_entry_t;

typedef struct {
    volatile uint32_t write_idx;  /* Producer only */
    volatile uint32_t read_idx;   /* Consumer only */
    ring_entry_t entries[RING_BUFFER_SIZE];
} ring_buffer_t;

CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static ring_buffer_t ring_buf;
```

**Write (CM55):** Check `next_write != read_idx`, copy entry, clean DCache, advance write_idx.
**Read (CM33):** Check `read_idx != write_idx`, invalidate DCache, copy entry, advance read_idx.

SPSC (single-producer single-consumer) — no lock needed. Each index owned by one core.

---

# Part 5: Cross-Core Transparent Printf

## Problem
When both CM33 and CM55 print to the same UART, output interleaves at the character level. This pattern serializes output through a shared ring buffer.

> **Note:** Both CM33 and CM55 can independently access their own UART SCB — this pattern is optional. Use it only when you need clean, non-interleaved output from both cores.

## Architecture
```
CM33 NS:  printf() → ring_write() → shared ring buffer (IPC locked)
CM55:     drain_task reads ring → UART output (via standard retarget-io)
CM55:     printf() → direct UART output (via standard retarget-io)
```

## Ring Buffer (shared header)

```c
#define DUAL_PRINTF_RING_SIZE  (2048U)
typedef struct {
    volatile uint32_t head, tail;
    volatile uint8_t ready; uint8_t _pad[3];
    char data[DUAL_PRINTF_RING_SIZE];
} dual_printf_ring_t;
```

Storage: `CY_SECTION_SHAREDMEM` in CM55 build only. CM33 gets address via IPC data register.

## IPC Lock (NOT PDL Semaphore)

```c
#define IPC_MUTEX_CHANNEL  (9U)
static IPC_STRUCT_Type *ipc_mutex_base;

static inline void mutex_acquire(void) {
    uint32_t timeout = 0;
    while (CY_IPC_DRV_SUCCESS != Cy_IPC_Drv_LockAcquire(ipc_mutex_base))
        if (++timeout > 100000UL) return;
}
static inline void mutex_release(void) {
    Cy_IPC_Drv_LockRelease(ipc_mutex_base, CY_IPC_NO_NOTIFICATION);
}
```

> **Never use `Cy_IPC_Sema_Init()`** — it overwrites `cy_semaIpcStruct`, breaking the BSP's SRF.

## Linker Hook + Drain Task

CM33 NS overrides `_write` to redirect printf to the shared ring buffer instead of its local UART. CM55 runs a drain task that reads the ring buffer and outputs to UART.

> **Note:** The `--wrap=_write` approach is one option for IPC-routed printf. An alternative is to use explicit logging macros (e.g., `LOG_CM33(...)`) that write to the ring buffer directly, avoiding linker hooks entirely.

## Build Roles

```makefile
# proj_cm55/Makefile:
DEFINES+=DUAL_PRINTF_ROLE_UART_OWNER

# proj_cm33_ns/Makefile:
DEFINES+=DUAL_PRINTF_ROLE_IPC_REMOTE
LDFLAGS+=-Wl,--wrap=_write
```

Only the IPC_REMOTE side (CM33 NS) needs the `--wrap=_write` flag — it redirects CM33's printf to the ring. CM55 prints directly to UART and also drains CM33's ring buffer.

---

# Part 6: Choosing the Right IPC Pattern

| Use case | Pattern | Throughput | Complexity |
|----------|---------|-----------|------------|
| State changes, commands | IPC Message Queue | Low (8 msg) | Low |
| Periodic sensor readings | Shared Memory + Magic | Medium | Medium |
| Streaming data (radar, audio) | Ring Buffer | High | Higher |
| Boot synchronization | IPC Handshake | N/A | Low |
| Debug printf from CM33 | Transparent Printf Ring | N/A | Medium |

## Common IPC Pitfalls

1. **PDL semaphore breaks SRF** — use raw IPC driver locks on dedicated channels
2. **D-cache stale reads** — use `CY_SECTION_SHAREDMEM` or explicit cache maintenance
3. **Ring buffer overflow** — bytes silently dropped; increase size or reduce volume
4. **Boot ordering** — CM55 must init ring/shared data before CM33 reads it
5. **IPC channel collision** — channels 0–7 typically reserved by BSP
6. **`__real__write` missing** — only exists on the core with `--wrap=_write` in LDFLAGS (IPC_REMOTE side only)
