# PSOC Edge Dual-Core Development Guide

A practical reference for building dual-core applications on PSOC Edge E84 (Cortex-M55 + Cortex-M33).

> Distilled from validated code examples CE-05 (Radar Presence), CE-06 (Radar Micro-Motion), and the Dual-Core Printf Reference project.

---

## Architecture

PSOC Edge E84 runs two application cores:
- **CM33** — Initialization core (TrustZone), application logic, WiFi/BT, MQTT
- **CM55** — High-performance compute: DSP, ML inference, radar processing (400 MHz, Arm Helium/MVE SIMD)

The 3-project layout is mandatory:
- `proj_cm33_s/` — TrustZone secure world (boilerplate)
- `proj_cm33_ns/` — All application code
- `proj_cm55/` — Deep sleep (default) or active DSP

---

## Boot Sequence

1. CM33 Secure boots first, configures TrustZone, launches CM33 NS
2. CM33 NS runs `cybsp_init()`, initializes retarget-io, then IPC
3. CM33 NS calls `Cy_SysPm_Cm55Enable(CY_CORTEX_M55_ENABLE)` to start CM55
4. CM33 NS sends `BOOT_READY` sentinel via IPC
5. CM55 polls `mtb_ipc_get_handle()` until non-null, then waits for sentinel
6. Both cores proceed with application initialization

**Critical:** CM55 must NOT call any IPC function before CM33 completes `mtb_ipc_init()`. A null IPC handle causes a HardFault with no diagnostic.

---

## IPC Subsystem

### Message format
- Fixed **16 bytes** per message
- Queue depth: **8 entries**
- Hardware semaphores guarantee atomicity
- Shared SRAM region — no mailbox interrupts needed

### Shared memory requirements
```c
CY_SECTION_SHAREDMEM __attribute__((aligned(32)))
static uint8_t ipc_buffer[BUFFER_SIZE];
```
- `CY_SECTION_SHAREDMEM` — places in correct linker section
- `aligned(32)` — DCache line size on CM55
- Without both attributes: sporadic data corruption from DCache writeback

### IPC design principles
1. **Event-driven, not streaming** — send state transitions, not every frame
2. **Never block the producer** — if queue is full, skip or buffer locally
3. **Holdoff timer** — minimum 100ms between sends prevents queue overflow
4. **Keep messages small** — 4 × uint32_t (type, param1, param2, param3)

---

## Printf on Dual-Core

Both cores share one physical UART (KitProg3). Simultaneous printf interleaves at character level.

### Solution: `--wrap=_write` pattern
PSOC Edge requires a linker wrapper for printf to work:
```makefile
LDFLAGS+=-Wl,--wrap=_write
```
See `#retarget-io-fix` prompt for the complete implementation.

### Core-prefix convention
```c
#define LOGM(fmt, ...) printf("[CM33] " fmt "\n", ##__VA_ARGS__)
#define LOGC(fmt, ...) printf("[CM55] " fmt "\n", ##__VA_ARGS__)
```

---

## CM55 FreeRTOS Configuration

When running FreeRTOS on CM55 with Helium/MVE:

```c
/* FreeRTOSConfig.h — CM55 */
#define configENABLE_MVE  1   /* MANDATORY for Helium SIMD */
```

Without this, FreeRTOS won't save MVE registers on context switch. Any task using SIMD intrinsics will silently corrupt data when preempted by another task.

---

## Helium/MVE Acceleration

CM55 supports Arm Helium (MVE) SIMD for DSP workloads.

### Compiler flags
```makefile
CFLAGS+=-march=armv8.1-m.main+mve.fp+fp.dp -mfloat-abi=hard
```

### CMSIS-DSP
Use CMSIS-DSP functions (`arm_*`) — they auto-vectorize with Helium when compiled correctly:
- `arm_rfft_fast_f32()` — Real FFT
- `arm_cmplx_mag_f32()` — Complex magnitude
- `arm_sub_f32()` — Vector subtraction
- `arm_max_f32()` — Vector maximum

Typical speedup: **4-5× over scalar** for FFT and vector operations.

### Toolchain comparison
| | GCC_ARM | LLVM |
|--|---------|------|
| Helium intrinsics | ✓ | ✓ |
| Auto-vectorization | Good | Better |
| Build speed | Faster | Slower |
| Recommended for | General use | DSP-heavy workloads |

---

## WiFi/MQTT on CM33

CM33 handles all network communication. Key patterns from validated examples:

### WiFi Connection
- Use `cy_wcm_connect_ap()` with retry loop (5 attempts, 2s delay)
- Always check `cy_wcm_is_connected_to_ap()` before MQTT operations
- Add a network monitor task for reconnection

### MQTT Publishing
- Rate-limit publishes to 1-2 per second (TLS encryption is CPU-intensive on CM33)
- Use QoS 1 for reliable delivery
- Handle disconnect events and reconnect

### TLS
- Store root CA cert as PEM string in source
- Use port 8883 for MQTT over TLS
- Mbed TLS handles the crypto — requires `COMPONENTS+=MBEDTLS`

---

## Build System Patterns

### common.mk (root level)
```makefile
TARGET=KIT_PSE84_EVAL_EPC2   # or KIT_PSE84_AI for radar kits
TOOLCHAIN=GCC_ARM
```

### Root Makefile
```makefile
MTB_TYPE=APPLICATION
MTB_PROJECTS=proj_cm33_s proj_cm33_ns proj_cm55
```
Build order is automatic: s → ns → cm55.

### Per-subproject deps/
Each subproject can have its own `deps/` folder with `.mtb` files. Libraries in `proj_cm33_ns/deps/` are only linked to CM33 NS. Libraries in `proj_cm55/deps/` are only linked to CM55.

---

## Common Mistakes Summary

| Mistake | Symptom | Fix |
|---------|---------|-----|
| App logic in `proj_cm33_s/` | TrustZone security fault | All app code in `proj_cm33_ns/` |
| CM55 IPC before CM33 init | HardFault (null handle) | Boot handshake pattern |
| Missing `CY_SECTION_SHAREDMEM` | Sporadic data corruption | Add section + alignment |
| Missing `configENABLE_MVE=1` | Silent SIMD corruption | Add to CM55 FreeRTOSConfig |
| No `--wrap=_write` | Printf produces no output | Add LDFLAGS wrapper |
| Streaming data over IPC | Queue overflow, CM55 stall | Event-driven + holdoff |
| TLS publish rate too high | MQTT disconnect | Rate limit 1-2/sec |
