# Dual-Core Setup — PSOC Edge E84

Set up a PSOC Edge project with an **active CM55 core** running application logic alongside CM33.

> **Prerequisites:** Read `CONTEXT.md` first. This prompt applies only to PSOC Edge E84 (3-project layout).

---

## When to use this prompt

- Your CM55 needs to run DSP, radar, audio, ML inference, or other compute-intensive workloads
- You need IPC (inter-processor communication) between CM33 and CM55
- Default `proj_cm55/main.c` deep-sleep loop needs to be replaced with active code

---

## Step 1: Project Structure

The 3-project layout is mandatory. CM55 gets its own `source/` and `include/`:

```
proj_cm55/
├── Makefile
├── main.c
├── source/
│   └── cm55_app_task.c      ← Your CM55 application logic
├── include/
│   └── cm55_app_task.h
├── FreeRTOSConfig.h          ← CM55 needs its own FreeRTOS config
└── deps/
    └── freertos.mtb          ← If using FreeRTOS on CM55
```

## Step 2: CM55 Makefile Configuration

Add to `proj_cm55/Makefile`:
```makefile
# Enable FreeRTOS on CM55
COMPONENTS+=FREERTOS

# If using Arm Helium/MVE SIMD instructions
DEFINES+=configENABLE_MVE=1
```

> **Critical:** Without `configENABLE_MVE=1`, FreeRTOS will not save MVE/Helium registers on context switch. Any task using SIMD intrinsics will silently corrupt data when preempted.

## Step 3: Boot Synchronization (CM33 side)

In `proj_cm33_ns/main.c`, CM33 must initialize IPC and enable CM55 **before** CM55 attempts any IPC:

```c
#include "mtb_ipc.h"
#include "cy_syspm.h"

/* Define IPC message structure */
typedef struct {
    uint32_t type;
    uint32_t data[3];  /* 16 bytes total — IPC message size limit */
} ipc_msg_t;

#define IPC_MSG_BOOT_READY  0x0001
#define IPC_TIMEOUT_MS      1000

/* Shared memory — must be in shared SRAM, aligned to DCache line */
CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static uint8_t ipc_shared_buffer[256];

void app_init(void)
{
    /* Initialize IPC subsystem */
    cy_rslt_t result = mtb_ipc_init();
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    /* Enable CM55 core — it will start executing proj_cm55/main.c */
    Cy_SysPm_Cm55Enable(CY_CORTEX_M55_ENABLE);

    /* Send boot-ready sentinel so CM55 knows IPC is ready */
    ipc_msg_t boot_msg = { .type = IPC_MSG_BOOT_READY };
    result = mtb_ipc_send(&boot_msg, sizeof(boot_msg), IPC_TIMEOUT_MS);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
}
```

## Step 4: Boot Synchronization (CM55 side)

In `proj_cm55/main.c`, CM55 waits for the IPC handshake before proceeding:

```c
#include "mtb_ipc.h"

void cm55_ipc_init(void)
{
    /* Wait for CM33 to initialize IPC — handle is null until ready */
    mtb_ipc_handle_t ipc_handle = NULL;
    while (ipc_handle == NULL)
    {
        ipc_handle = mtb_ipc_get_handle();
        Cy_SysLib_Delay(1);  /* 1 ms poll interval */
    }

    /* Wait for boot-ready sentinel from CM33 */
    ipc_msg_t rx_msg = {0};
    while (rx_msg.type != IPC_MSG_BOOT_READY)
    {
        mtb_ipc_receive(ipc_handle, &rx_msg, sizeof(rx_msg), IPC_TIMEOUT_MS);
    }

    printf("[CM55] Boot handshake complete\n");
}
```

> **Warning:** If CM55 calls `mtb_ipc_get_handle()` before CM33 calls `mtb_ipc_init()`, it returns NULL. Attempting to use a NULL handle causes a HardFault with no diagnostic output.

## Step 5: Shared Memory Requirements

IPC shared memory buffers must satisfy two requirements:

1. **Section attribute:** `CY_SECTION_SHAREDMEM` places the buffer in the correct linker section accessible to both cores
2. **Alignment:** `__attribute__((aligned(32)))` aligns to DCache line size (32 bytes on CM55)

```c
/* Correct — both attributes required */
CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static uint8_t shared_data[SHARED_DATA_SIZE];

/* WRONG — will cause sporadic data corruption from DCache writeback */
static uint8_t shared_data[SHARED_DATA_SIZE];
```

## Step 6: IPC Message Design

IPC messages are fixed at **16 bytes**, queue depth is **8 entries**. Design for state-change events, not streaming data:

```c
typedef struct {
    uint32_t type;       /* Message type enum */
    uint32_t timestamp;  /* System tick or frame counter */
    uint32_t value;      /* Primary data (float reinterpret as uint32 if needed) */
    uint32_t reserved;   /* Pad to 16 bytes */
} ipc_msg_t;
```

**Key rule:** Send on state transitions (e.g., IDLE→ACTIVE), not on every frame. A holdoff timer (min 100ms between sends) prevents queue overflow when the consumer (usually CM33 with WiFi/MQTT) can't keep up.

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| CM55 calls `mtb_ipc_get_handle()` before CM33 `mtb_ipc_init()` | HardFault on first IPC send | Add boot handshake (Steps 3-4) |
| Missing `CY_SECTION_SHAREDMEM` on shared buffer | Sporadic data corruption | Add section + alignment attributes |
| Missing `configENABLE_MVE=1` in CM55 FreeRTOSConfig | Silent SIMD data corruption after task switch | Add to FreeRTOSConfig.h or Makefile DEFINES |
| Sending IPC message every DSP frame | Queue overflow, CM55 stalls | Send state changes only + holdoff timer |
| App logic in `proj_cm33_s/` | TrustZone security violations | All app code in `proj_cm33_ns/` |
