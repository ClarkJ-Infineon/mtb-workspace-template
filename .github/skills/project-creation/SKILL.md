---
name: project-creation
description: Non-negotiable workflow for creating ModusToolbox projects. Use this when creating a new project, discussing project setup, when build errors suggest missing BSP or generated code infrastructure, or when someone asks how to start a new ModusToolbox project.
---

# Project Creation — ModusToolbox™ Workflow Rule

> **This is a strict workflow rule, not a suggestion.**
> Every ModusToolbox™ project MUST be created via `project-creator-cli`. Manual project assembly (cloning repos, copying BSP files, hand-writing Makefiles) is not a supported workflow and will produce projects with hidden defects.

Read `CONTEXT.md` first for current software versions, target device, and kit.

---

## Why This Rule Exists

The ModusToolbox™ build system depends on generated infrastructure that only `project-creator-cli` produces correctly:

| Generated artifact | What it provides | What breaks without it |
|---|---|---|
| `bsps/TARGET_*/` | Board Support Package matched to your SDK version | Wrong HAL APIs, missing pin definitions, linker script mismatches |
| `design.modus` personalities | Device Configurator peripheral configurations | `cycfg_peripherals.h` missing — all GFXSS, I2C, UART symbols undefined |
| `cycfg_*.c/h` generated code | IRQ routing, peripheral structures, clock trees, pin mux | Build fails with dozens of undefined symbol errors; cannot be fixed manually |
| `libs/mtb.mk` | `$(SEARCH_*)` variables for all library include paths | Silent include path failures — headers not found, no error message |
| `mtb_shared/` | Version-locked shared library dependencies | Version drift between libraries causes runtime crashes |
| Multi-core linker scripts | Coordinated memory maps across CM33 S / CM33 NS / CM55 | IPC corruption, HardFaults, cores addressing different physical memory |

**Evidence:** The ce-weather-display project lost ~15 hours (4 sessions) to manual project assembly. Five of seven integration failures traced directly to not using `project-creator-cli`. After pivoting to a project-creator-cli-generated project, the same functionality was achieved in one session.

---

## The Rule

### For New Projects

```bash
# Always start here — no exceptions
project-creator-cli \
    --board-id KIT_PSE84_EVAL_EPC2 \
    --app-id <template-id> \
    --target-dir <output-dir> \
    --user-app-name <project-name>
```

**Choosing the starting template:**

| Your goal | Recommended template | Rationale |
|---|---|---|
| WiFi / MQTT / HTTP / cloud | Empty app or WiFi template | Add connectivity libs via /add-library skill |
| BLE | Empty app or BLE template | Add BLE stack via /ble-setup skill |
| LVGL graphics | LVGL demo app (`mtb-example-psoc-edge-lvgl-demo`) | GFXSS/GPU/display stack requires Device Configurator personalities that only this template provides |
| Dual-core with active CM55 | Empty app | Configure CM55 via /dual-core-setup skill |
| Sensor / motor / bare-metal | Empty app | Add peripherals via /device-configurator-spec and /add-library skills |
| Graphics + connectivity | LVGL demo app, then add connectivity | Graphics is the hardest subsystem to bootstrap; connectivity is additive. See §Building Up From Empty below. |

**To list available templates:**
```bash
project-creator-cli --list-apps
```

**To list available boards:**
```bash
project-creator-cli --list-boards
```

### For Existing Projects (Adding Capabilities)

If your project was already created with `project-creator-cli`, you can add capabilities incrementally:

1. **Add libraries** → /add-library skill (creates `.mtb` files, updates Makefile)
2. **Add peripherals** → /device-configurator-spec skill (documents Device Configurator settings)
3. **Add dual-core** → /dual-core-setup skill (CM55 activation, IPC, boot sync)
4. **Add WiFi/MQTT** → /wifi-mqtt skill (connection manager, MQTT client)
5. **Add BLE** → /ble-setup skill (BTSTACK, GATT services)
6. **Add LVGL graphics** → /lvgl-setup skill (GFXSS personality, display drivers, VG-Lite)

Each skill is designed to layer onto an existing `project-creator-cli`-generated project. They assume the base infrastructure (BSP, linker scripts, `design.modus`) is correct.

---

## Building Up From Empty — The Goal

The practical advice today is: **start from the template closest to your goal**. For graphics, that means the LVGL demo. For WiFi, the WiFi template. This is the fastest path to a working project.

But the strategic goal is different: **it should always be possible to start from an empty app and build up to any project** by composing the right skills. Each skill encodes one capability domain:

```
Empty App (via project-creator-cli)
  + /dual-core-setup          → Active CM55
  + /add-library               → Any middleware
  + /wifi-mqtt                 → WiFi + MQTT
  + /ble-setup                 → Bluetooth LE
  + /lvgl-setup                → Graphics / touch display
  + /ipc-patterns              → Cross-core data sharing
  + /device-configurator-spec  → Any peripheral
```

Where this "build-up" path is **proven and documented**, use it with confidence. Where it is **not yet validated** (marked below), start from the closest template and file a gap for the missing skill.

### Capability Readiness Matrix

| Capability | Skill | Status | Notes |
|---|---|---|---|
| Dual-core CM55 activation | /dual-core-setup | ✅ Validated | Boot sync, FreeRTOS, IPC init |
| WiFi + MQTT | /wifi-mqtt | ✅ Validated | Super-dep bundle, TLS, reconnect |
| BLE | /ble-setup | ✅ Validated | BTSTACK v4 API via v6 integration |
| Library addition | /add-library | ✅ Validated | .mtb format, COMPONENTS, DEFINES |
| Retarget-io (printf) | /retarget-io-fix | ✅ Validated | PSOC Edge wrapper pattern |
| IPC shared memory | /ipc-patterns | ⚠️ Needs update | Lock-free pattern from ce-weather-display not yet incorporated |
| LVGL graphics | /lvgl-setup | ✅ Created | GFXSS, display drivers, VG-Lite, lv_conf.h, init sequence |
| HTTP client | /http-client | ✅ Created | cy_secure_sockets, coreJSON, polling pattern |
| CM55 transparent printf | /transparent-printf | ✅ Created | Ring buffer IPC relay, --wrap=_write, drain task |
| OpenOCD/GDB debugging | /openocd-debug | ✅ Created | make debug/attach, multi-core, CFSR analysis |
| Radar DSP | /radar-dsp | ✅ Validated | CM55 Helium/MVE, FFT pipelines |

As skills are created and validated, the "start from closest template" advice becomes less necessary. The end state: **any developer (human or AI) can start from an empty app and compose capabilities via skills.**

---

## What Goes Wrong Without `project-creator-cli`

### Anti-pattern 1: Clone and manually configure
```bash
# ❌ NEVER DO THIS
git clone https://github.com/Infineon/TARGET_KIT_PSE84_EVAL_EPC2
# then manually copy files, write Makefiles, etc.
```
**Result:** Missing `design.modus` personalities, no generated code, `$(SEARCH_*)` variables undefined, linker scripts not coordinated across cores.

### Anti-pattern 2: Copy from a "working project" on GitHub
```bash
# ❌ DANGEROUS — version drift, stale APIs, non-standard patterns
# Even if the project builds, its BSP may be stale
```
**Result:** API mismatches (VG-Lite signatures, cybsp_init() return types), cached build artifacts that mask dependency issues, non-standard Makefile patterns that break on update.

### Anti-pattern 3: `make getlibs` without Project Creator import
```bash
# ❌ INCOMPLETE — does not resolve BSP or base infrastructure
git clone <ce-template-repo>
cd <project>
make getlibs
make build   # FAILS
```
**Result:** `make getlibs` only downloads libraries listed in `.mtb` files. It does not create the BSP, generate Device Configurator output, or set up `mtb_shared/`. CE templates are designed for Project Creator import, not standalone `make getlibs`.

---

## For AI Agents

If you are an AI coding agent working on a ModusToolbox™ project:

1. **Never create a project by writing Makefiles and source files from scratch.** Generate a template that the developer imports via `project-creator-cli`.
2. **Never copy BSP files, `design.modus`, or `cycfg_*.c/h` from another project.** These are generated artifacts tied to a specific SDK version and board configuration.
3. **Never assume a "working example" found on GitHub is a valid reference.** Verify it is an official Infineon code example at the current SDK version before using any of its code.
4. **When a build fails with undefined symbols** (`GFXSS_Type`, `Cy_SCB_*`, `CYBSP_*`), check whether the project was created via `project-creator-cli` before debugging further. The most common cause is missing generated code.
5. **When composing capabilities**, use the skills in this workspace. Each skill encodes validated patterns that work with `project-creator-cli`-generated projects.

---

## Reference

| Resource | Location |
|---|---|
| Project Creator CLI docs | `ModusToolbox/tools_<version>/docs/project-creator-cli.html` (check CONTEXT.md for version) |
| Available templates | `project-creator-cli --list-apps` |
| Available boards | `project-creator-cli --list-boards` |
