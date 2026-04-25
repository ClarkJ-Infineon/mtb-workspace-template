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

    /* CRITICAL: Invalidate D-Cache for shared memory after cybsp_init().
     * Reset_Handler() enables D-Cache BEFORE cybsp_init() configures MPU
     * non-cacheable regions. Stale cache lines from that boot window persist
     * and will return wrong values on first read of IPC/shared memory. */
    SCB_InvalidateDCache_by_Addr((void *)IPC_SHARED_ADDR, sizeof(ipc_shared_t));

    while (!cm33_ready_flag) { __WFE(); }
    vTaskStartScheduler();
}
```

## Boot-Time D-Cache Invalidation (CM55) — CRITICAL

On PSOC Edge, the CM55 `Reset_Handler()` (in `ns_start_pse84.c`) calls `SCB_EnableDCache()` **before** `main()` runs. MPU non-cacheable regions are not configured until `cybsp_init()` → `Cy_MPU_Init()` inside `main()`. During this window, any memory access to SOCMEM (IPC shared memory, framebuffers) creates cacheable cache lines.

After `cybsp_init()` reconfigures MPU to make SOCMEM non-cacheable, those **stale cache lines persist** — the MPU change does not flush existing lines. Subsequent reads return boot-time stale data, not the actual memory contents.

**Fix:** Immediately after `cybsp_init()`, invalidate D-Cache for all shared-memory regions:

```c
cybsp_init();
/* Flush stale D-Cache lines created before MPU was configured */
SCB_InvalidateDCache_by_Addr((void *)IPC_SHARED_ADDR, sizeof(ipc_shared_t));
/* If using graphics framebuffers, also invalidate gfx_mem region */
```

> **`volatile` does NOT help here.** The `volatile` keyword prevents compiler optimizations but does NOT bypass the hardware D-Cache. Only MPU non-cacheable configuration + D-Cache invalidation solves this.

## Common Mistakes — Core Setup

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `configENABLE_MVE=1` | Silent data corruption in SIMD tasks | Add to CM55 Makefile DEFINES |
| CM55 starts before CM33 ready | HardFault or IPC deadlock on boot | Add boot sync handshake |
| No `FreeRTOSConfig.h` in `proj_cm55/` | CM55 uses CM33's config (wrong sizes) | Create separate config |
| CM55 tries UART directly | No output (PPU blocks SCB2) | Use IPC printf relay (Part 4) |
| No D-Cache invalidation after `cybsp_init()` | IPC reads return stale/zero data at boot | Add `SCB_InvalidateDCache_by_Addr()` (see above) |

## Peripheral-to-Core Assignment

Not all peripherals are accessible from both cores. Use these rules to decide which core owns which peripheral:

### PPU (Protection Policy Unit) Constraints

The PPU restricts CM55 from accessing certain CM33-reserved peripherals. A PPU violation = BusFault with BFAR pointing to the peripheral address. **No runtime error message** — just a silent fault.

- **WiFi/BLE radio (CYW55500/CYW55513):** Accessible from BOTH CM33 and CM55 — official Infineon WiFi examples run on either core. There is no PPU restriction on the radio peripheral.
- **Debug UART (SCB2):** CM33 only by default — CM55 needs the IPC printf relay (Part 4) for output

### Shared Bus Awareness

**If two peripherals share the same I2C/SPI bus (SCB instance), they must be managed by the same core.** There is no hardware arbitration between cores on a shared SCB — simultaneous access from CM33 and CM55 will corrupt transactions.

Common scenario: An EVK may expose I2C pins on Arduino headers that share the same SCB instance used by an on-board peripheral (e.g., a display touch controller). Any external sensor connected to those shared pins must be driven by the same core that owns the on-board peripheral.

**How to check:** Open Device Configurator → Peripherals → examine which SCB instances are assigned and what they connect to. Each SCB can only be safely owned by one core.

### SCB Ownership Rule

**Only ONE core may own a given SCB instance at a time.** There is no hardware arbitration. If CM33 was using an SCB for a sensor that shares the bus with a CM55-owned peripheral (display/touch), the sensor must migrate to CM55 or use a different SCB.

> **Example (Matter Thermostat):** BME280 originally ran on CM33 via SCB0. The LVGL touchscreen also uses SCB0 on CM55. Solution: move BME280 to CM55 so both peripherals share the same core's SCB driver.

### General Assignment Guidance

| Capability | Typical Core | Reason |
|---|---|---|
| WiFi, BLE, MQTT, cloud connectivity | Either (CM33 or CM55) | Both cores can access the radio; choose based on overall app architecture |
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
7. **`ipc_manager_init()` zeroes shared memory** — call before any IPC data writes (see Part 7)
8. **IPC poll task must wait for app readiness** — use a readiness flag, not a fixed delay (see Part 7)
9. **`volatile` does NOT bypass D-Cache** — use MPU non-cacheable regions for SOCMEM (see Part 7)
10. **Factory reset from IPC task** — route through app event queue; IPC task stack too small (see Part 7)
11. **SCB ownership** — only one core may own a given SCB instance; migrate peripherals if buses collide (see Part 1)

---

# Part 7: IPC Shared Memory — Lessons Learned (Matter Thermostat + LVGL)

Hard-won patterns from integrating a dual-core Matter thermostat with LVGL display on CM55.

## IPC Init Ordering (CRITICAL)

`ipc_manager_init()` zeroes the entire shared struct with `memset()`. If any data has been written to shared memory **before** init runs (e.g., QR code payload written during Matter server init), it gets wiped.

> **Always ensure `ipc_manager_init()` runs before any IPC data writes.** In the Matter thermostat, `ipc_manager_init()` had to be moved before `chip::Server::Init()` because the QR code is generated during server init and written to IPC.

## IPC Poll Task Must Wait for Application Readiness

If the IPC poll task starts before the application layer (Matter stack, etc.) is fully initialized, commands like factory reset can cause crashes. Use a readiness flag set after init completes:

```c
static volatile bool s_matter_ready = false;

void ipc_poll_task(void *arg)
{
    /* Spin until the application layer signals readiness */
    while (!s_matter_ready) {
        vTaskDelay(pdMS_TO_TICKS(100));
    }

    for (;;) {
        ipc_manager_poll();
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

> **Do NOT use a fixed delay (e.g., 5 seconds)** — application init time varies by build config, network state, and flash contents.

## D-Cache and Shared Memory

IPC shared memory in SOCMEM must be **non-cacheable** via MPU on **BOTH** cores. On PSOC Edge, SOCMEM (`0x26000000` range) defaults to normal cacheable memory. Configure an MPU region or place shared memory in a known non-cacheable region.

> **The `volatile` keyword does NOT bypass D-Cache on ARM Cortex-M** — it only prevents compiler reordering/elimination. Without MPU non-cacheable configuration, reads may return stale cache lines even with `volatile`.

Alternative: use explicit cache maintenance (`SCB_CleanDCache_by_Addr` / `SCB_InvalidateDCache_by_Addr`) on every access, but MPU non-cacheable is simpler and less error-prone.

## Command One-Shot Pattern (CM55 → CM33)

For commands that cross cores, use a `cmd_pending` flag pattern that avoids mutexes:

```c
/* In shared memory struct */
typedef struct {
    volatile uint32_t cmd_type;
    volatile uint32_t cmd_value;
    volatile uint32_t cmd_pending;  /* 1 = new command, 0 = processed */
} ipc_command_t;
```

**Writer (CM55):**
```c
shared->cmd_type  = CMD_FACTORY_RESET;
shared->cmd_value = 0;
__DMB();                     /* Ensure writes are visible before flag */
shared->cmd_pending = 1;
```

**Reader (CM33):**
```c
if (shared->cmd_pending) {
    __DMB();                 /* Ensure flag read before data reads */
    uint32_t type  = shared->cmd_type;
    uint32_t value = shared->cmd_value;
    shared->cmd_pending = 0; /* Acknowledge */
    process_command(type, value);
}
```

> This avoids lost commands without requiring mutexes. The `__DMB()` barriers ensure correct ordering.

## IPC Command Decoupling by Framework Dependency

When CM55 sends commands via IPC, some can be processed immediately (mode changes, setpoint updates) while others require the application framework (Matter, MQTT, etc.) to be fully initialized (factory reset, commissioning state changes). Use a two-tier command handler:

```c
/* Process commands that don't need Matter — safe to call from boot */
void ipc_process_commands_local(ipc_shared_t *shared) {
    if (!shared->cmd_pending) return;
    switch (shared->cmd_type) {
        case CMD_SET_MODE:
        case CMD_SET_HEAT_SP:
        case CMD_SET_COOL_SP:
            apply_local_state(shared->cmd_type, shared->cmd_value);
            shared->cmd_pending = 0;
            break;
        case CMD_FACTORY_RESET:
            /* Leave pending — will be processed after framework init */
            break;
    }
}

/* Full command processing — only call after framework is ready */
void ipc_process_commands(ipc_shared_t *shared) {
    if (!shared->cmd_pending) return;
    switch (shared->cmd_type) {
        case CMD_SET_MODE:
        case CMD_SET_HEAT_SP:
        case CMD_SET_COOL_SP:
            apply_and_sync_to_framework(shared->cmd_type, shared->cmd_value);
            shared->cmd_pending = 0;
            break;
        case CMD_FACTORY_RESET:
            post_factory_reset_event();  /* Route through event queue */
            shared->cmd_pending = 0;
            break;
    }
}
```

> **Why:** Starting IPC polling immediately gives the user a responsive UI from boot, even while WiFi/Matter/BLE are still initializing (which can take 10+ seconds). Framework-dependent commands are deferred, not dropped.

## Setpoint Crossover Enforcement (Thermostat Pattern)

When an external source (Home App, IPC command) sets one setpoint past the opposing setpoint, auto-adjust the opposing setpoint to maintain the minimum deadband:

```c
#define DEADBAND_X100  250  /* 2.5°F in 0.01°C units */

void enforce_deadband(int16_t *heat_sp, int16_t *cool_sp) {
    if (*cool_sp < *heat_sp + DEADBAND_X100) {
        /* Adjust the one that WASN'T just changed */
        /* Caller passes which was the "active" setpoint */
    }
}
```

> **Context:** Matter thermostat clusters allow independent heat/cool setpoint writes. The Home App may set a cool setpoint below the current heat setpoint. Without enforcement, the UI shows an impossible state. The device (CM33 authority) must normalize and republish corrected values via IPC so the display and Home App both see consistent state.

## Factory Reset via Event Queue

Never call `chip::Server::GetInstance().ScheduleFactoryReset()` directly from the IPC task — the IPC poll task stack may be too small (512 bytes). Route through the application's event queue which has a larger stack:

```c
/* In IPC command handler — DO NOT call ScheduleFactoryReset() here */
case CMD_FACTORY_RESET:
    AppEvent event;
    event.Type = AppEvent::kEventType_FactoryReset;
    PostEvent(&event);  /* Route through AppTask event queue */
    break;
```

## Sequence Counter for Freshness

The writer (typically CM33) increments a sequence counter on every state publish. The reader (typically CM55) skips processing if the sequence hasn't changed:

```c
/* Writer side — publish state */
memcpy(&shared->state, &new_state, sizeof(new_state));
__DMB();                      /* Release barrier — data before sequence */
shared->sequence++;

/* Reader side — consume state */
uint32_t seq = shared->sequence;
__DMB();                      /* Acquire barrier — sequence before data */
if (seq == last_seen_seq) return;  /* No new data */
memcpy(&local_state, &shared->state, sizeof(local_state));
last_seen_seq = seq;
```

---

# Part 8: UART Sharing Between Cores

## Safe Sharing

Both CM33 and CM55 can safely share the debug UART (SCB2). Output interleaves at the byte level but does not crash or corrupt data.

## CRITICAL: Always Init retarget-io on CM55

**ALWAYS call `init_retarget_io()` on CM55** even when CM33 uses the same UART. The retarget-io library's `_write()` override includes a mutex that asserts `cy_retarget_io_mutex_initialized`. Without init, any `printf()` on CM55 causes `CY_HALT()`.

```c
/* CM55 main.c — REQUIRED even if CM33 also uses the same UART */
cybsp_init();
init_retarget_io();  /* Must be called — initializes the printf mutex */
```

## Core-Prefixed Output

Prefix all log lines for disambiguation in serial output:

```c
/* CM33 */
printf("[CM33] Matter stack initialized\n");

/* CM55 */
printf("[CM55] LVGL display ready\n");
```

---

# Part 9: PSOC Edge HSIOM Register Layout (Diagnostic Trap)

## MXS22IOSS vs MXS40IOSS

PSOC Edge (MXS22IOSS) uses **8 bits per pin** in HSIOM `PORT_SEL` registers — NOT 4 bits like PSOC 6 (MXS40IOSS).

| Platform | IOSS IP | Bits per pin | PORT_SEL0 covers | PORT_SEL1 covers |
|----------|---------|-------------|-------------------|-------------------|
| PSOC 6 | MXS40IOSS | 4 bits | Pins 0–7 | N/A (single register) |
| PSOC Edge | MXS22IOSS | 8 bits | Pins 0–3 only | Pins 4–7 |

## Rule

**NEVER hardcode 4-bit HSIOM assumptions.** Always use the PDL accessor:

```c
/* Correct — works on both PSOC 6 and PSOC Edge */
en_hsiom_sel_t current = Cy_GPIO_GetHSIOM(port_base, pin_num);

/* WRONG — assumes 4-bit layout, fails silently on PSOC Edge */
uint32_t sel = (HSIOM->PORT_SEL[port] >> (pin * 4)) & 0xF;
```

> **Symptom:** Pins appear stuck in GPIO mode or mapped to the wrong peripheral. HSIOM reads return garbage because the register layout changed.
