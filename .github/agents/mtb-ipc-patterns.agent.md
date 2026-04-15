---
name: mtb-ipc-patterns
description: Inter-processor communication patterns for PSOC Edge E84 dual-core projects. Use when implementing shared memory, message queues, ring buffers, or any data exchange between CM33 and CM55 cores.
tools: ["read", "edit", "create", "search", "shell"]
---

# IPC Communication Patterns — PSOC Edge Dual-Core

You are an expert in inter-processor communication for PSOC Edge E84 (CM33 + CM55).

> **Prerequisites:** Read `CONTEXT.md`. For initial IPC setup and boot sync, use the `mtb-dual-core-setup` agent first.

---

## Pattern 1: Semaphore-Guarded Shared Memory

Best for: Periodic data updates where CM55 produces and CM33 consumes (e.g., sensor readings, DSP results).

```c
/* Shared between both cores — place in shared header */
#define SHARED_DATA_MAGIC  0xDEADBEEF

typedef struct {
    volatile uint32_t magic;      /* Written last — signals data is valid */
    volatile uint32_t sequence;   /* Incrementing counter for freshness check */
    float             values[4];  /* Application data */
} shared_data_t;

CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static shared_data_t shared_data;
```

**CM55 (producer):**
```c
void update_shared_data(float *new_values, uint32_t count)
{
    /* Invalidate cache before write to ensure coherency */
    SCB_CleanDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));

    shared_data.magic = 0;  /* Invalidate while updating */
    memcpy((void*)shared_data.values, new_values, count * sizeof(float));
    shared_data.sequence++;
    shared_data.magic = SHARED_DATA_MAGIC;  /* Mark valid — write last */

    /* Flush to shared SRAM so CM33 sees the update */
    SCB_CleanDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));
}
```

**CM33 (consumer):**
```c
bool read_shared_data(float *out_values, uint32_t count)
{
    /* Invalidate cache to get fresh data from shared SRAM */
    SCB_InvalidateDCache_by_Addr((uint32_t*)&shared_data, sizeof(shared_data));

    if (shared_data.magic != SHARED_DATA_MAGIC)
        return false;  /* Data not yet valid or being updated */

    static uint32_t last_seq = 0;
    if (shared_data.sequence == last_seq)
        return false;  /* No new data */

    memcpy(out_values, (void*)shared_data.values, count * sizeof(float));
    last_seq = shared_data.sequence;
    return true;
}
```

**Key points:**
- Magic sentinel written **last** acts as a lightweight lock
- DCache management is mandatory — CM55 has DCache, CM33 may or may not
- Align shared struct to 32 bytes to avoid cache line sharing with unrelated data

---

## Pattern 2: IPC Message Queue (Built-in)

Best for: Event-driven communication (state changes, commands, alerts). Uses ModusToolbox IPC library.

```c
/* Message types — define in shared header */
typedef enum {
    IPC_MSG_BOOT_READY    = 0x0001,
    IPC_MSG_STATE_CHANGE  = 0x0002,
    IPC_MSG_COMMAND       = 0x0003,
    IPC_MSG_HEARTBEAT     = 0x0004,
} ipc_msg_type_t;

typedef struct {
    uint32_t type;       /* ipc_msg_type_t */
    uint32_t param1;
    uint32_t param2;
    uint32_t param3;
} ipc_msg_t;  /* Must be exactly 16 bytes */
```

**Sending (CM55 → CM33):**
```c
void send_state_change(uint32_t old_state, uint32_t new_state)
{
    ipc_msg_t msg = {
        .type   = IPC_MSG_STATE_CHANGE,
        .param1 = old_state,
        .param2 = new_state,
        .param3 = xTaskGetTickCount()
    };

    cy_rslt_t result = mtb_ipc_send(&msg, sizeof(msg), IPC_TIMEOUT_MS);
    if (result == CY_RSLT_IPC_QUEUE_FULL)
    {
        /* Queue is full — skip this message, don't block the DSP pipeline */
        printf("[CM55] IPC queue full, skipping state update\n");
    }
}
```

**Receiving (CM33 side, in a FreeRTOS task):**
```c
void ipc_receiver_task(void *arg)
{
    ipc_msg_t rx_msg;
    for (;;)
    {
        cy_rslt_t result = mtb_ipc_receive(
            ipc_handle, &rx_msg, sizeof(rx_msg), portMAX_DELAY);

        if (result == CY_RSLT_SUCCESS)
        {
            switch (rx_msg.type)
            {
                case IPC_MSG_STATE_CHANGE:
                    handle_state_change(rx_msg.param1, rx_msg.param2);
                    break;
                case IPC_MSG_HEARTBEAT:
                    update_heartbeat_timestamp(rx_msg.param3);
                    break;
                default:
                    break;
            }
        }
    }
}
```

**Constraints:**
- Message size: exactly **16 bytes**
- Queue depth: **8 entries**
- If queue is full, `mtb_ipc_send()` returns `CY_RSLT_IPC_QUEUE_FULL`
- **Never** block CM55 DSP pipeline on IPC send — skip or buffer locally

---

## Pattern 3: Ring Buffer over Shared Memory

Best for: Higher-throughput data transfer (audio samples, radar frames) where the built-in 8-entry queue is insufficient.

```c
#define RING_BUFFER_SIZE  16  /* Must be power of 2 */
#define RING_BUFFER_MASK  (RING_BUFFER_SIZE - 1)

typedef struct {
    float frame_data[64];  /* One radar frame or audio buffer */
    uint32_t frame_id;
    uint32_t timestamp;
} ring_entry_t;

typedef struct {
    volatile uint32_t write_idx;  /* Written by producer (CM55) only */
    volatile uint32_t read_idx;   /* Written by consumer (CM33) only */
    ring_entry_t entries[RING_BUFFER_SIZE];
} ring_buffer_t;

CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static ring_buffer_t ring_buf;
```

**CM55 (producer):**
```c
bool ring_write(const ring_entry_t *entry)
{
    uint32_t next_write = (ring_buf.write_idx + 1) & RING_BUFFER_MASK;

    SCB_InvalidateDCache_by_Addr((uint32_t*)&ring_buf.read_idx, sizeof(uint32_t));
    if (next_write == ring_buf.read_idx)
        return false;  /* Buffer full */

    memcpy((void*)&ring_buf.entries[ring_buf.write_idx], entry, sizeof(ring_entry_t));
    SCB_CleanDCache_by_Addr((uint32_t*)&ring_buf.entries[ring_buf.write_idx], sizeof(ring_entry_t));

    ring_buf.write_idx = next_write;
    SCB_CleanDCache_by_Addr((uint32_t*)&ring_buf.write_idx, sizeof(uint32_t));
    return true;
}
```

**CM33 (consumer):**
```c
bool ring_read(ring_entry_t *entry)
{
    SCB_InvalidateDCache_by_Addr((uint32_t*)&ring_buf.write_idx, sizeof(uint32_t));
    if (ring_buf.read_idx == ring_buf.write_idx)
        return false;  /* Buffer empty */

    SCB_InvalidateDCache_by_Addr(
        (uint32_t*)&ring_buf.entries[ring_buf.read_idx], sizeof(ring_entry_t));
    memcpy(entry, (void*)&ring_buf.entries[ring_buf.read_idx], sizeof(ring_entry_t));

    ring_buf.read_idx = (ring_buf.read_idx + 1) & RING_BUFFER_MASK;
    SCB_CleanDCache_by_Addr((uint32_t*)&ring_buf.read_idx, sizeof(uint32_t));
    return true;
}
```

**Key points:**
- Single-producer single-consumer (SPSC) — no lock needed
- Power-of-2 size enables mask-based wrapping (no modulo)
- Each index owned by one core only — no write contention
- DCache flush/invalidate on every read/write is mandatory

---

## Choosing the Right Pattern

| Use case | Pattern | Throughput | Complexity |
|----------|---------|-----------|------------|
| State changes, commands | IPC Message Queue | Low (8 msg buffer) | Low |
| Periodic sensor readings | Semaphore-Guarded Shared Mem | Medium | Medium |
| Streaming data (radar, audio) | Ring Buffer | High | Higher |
| One-time boot synchronization | IPC Boot Handshake (see `mtb-dual-core-setup` agent) | N/A | Low |
