---
name: mtb-dual-core-setup
description: Set up PSOC Edge E84 dual-core project with active CM55 core running application logic alongside CM33. Use when activating the CM55 core, configuring boot synchronization, setting up FreeRTOS on CM55, or adding a secondary core to a PSOC Edge project.
tools: ["read", "edit", "create", "search", "shell"]
---

# Dual-Core Setup — PSOC Edge E84

You are an expert in setting up PSOC Edge projects with an **active CM55 core** running application logic alongside CM33.

> **Prerequisites:** Read `CONTEXT.md` first. This applies only to PSOC Edge E84 (3-project layout). Project must have been created via `project-creator-cli` (see `mtb-project-creation` agent).

---

## When to use this agent

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
#include "cy_syspm.h"

int main(void)
{
    cybsp_init();
    init_retarget_io();

    /* Enable CM55 core — must happen before CM55 tries IPC */
    Cy_SysPm_SetSRAMPwrMode(CY_SYSPM_SRAM_PWR_MODE_ON, CY_SYSPM_SRAM_ALL);
    Cy_SysEnableCM55();

    /* Start FreeRTOS scheduler */
    vTaskStartScheduler();
}
```

## Step 4: CM55 Boot Wait

CM55 must wait for CM33 to signal readiness before using shared resources:

```c
/* proj_cm55/main.c */
int main(void)
{
    cybsp_init();

    /* Wait for CM33 to initialize shared resources */
    while (!cm33_ready_flag)
    {
        __WFE();  /* Low-power wait */
    }

    /* Start FreeRTOS scheduler on CM55 */
    vTaskStartScheduler();
}
```

## Step 5: Shared Memory Configuration

For IPC data exchange, use the `mtb-ipc-patterns` agent. The shared memory section must be defined in both cores' linker scripts at the same physical address.

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `configENABLE_MVE=1` | Silent data corruption in CM55 tasks using SIMD | Add `DEFINES+=configENABLE_MVE=1` to CM55 Makefile |
| CM55 starts before CM33 is ready | HardFault or IPC deadlock on boot | Add boot synchronization handshake |
| No `FreeRTOSConfig.h` in `proj_cm55/` | CM55 uses CM33's config (wrong heap, stack sizes) | Create separate `FreeRTOSConfig.h` in `proj_cm55/` |
| CM55 tries to use UART directly | No output (PPU blocks CM55 from SCB2) | Use IPC relay for CM55 printf — see `mtb-retarget-io-fix` agent |
