---
name: mtb-project-creation
description: Non-negotiable workflow for creating ModusToolbox projects. Use this when creating a new project, discussing project setup, when build errors suggest missing BSP or generated code infrastructure, or when someone asks how to start a new ModusToolbox project.
tools: ["read", "edit", "create", "search", "shell"]
---

# Project Creation — ModusToolbox™ Workflow Rule

You enforce the strict rule that every ModusToolbox™ project MUST be created via `project-creator-cli`. Manual project assembly is not a supported workflow.

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
| WiFi / MQTT / HTTP / cloud | Empty app or WiFi template | Add connectivity libs via `mtb-add-library` agent |
| BLE | Empty app or BLE template | Add BLE stack via BLE setup |
| LVGL graphics | LVGL demo app (`mtb-example-psoc-edge-lvgl-demo`) | GFXSS/GPU/display stack requires Device Configurator personalities that only this template provides |
| Dual-core with active CM55 | Empty app | Configure CM55 via `mtb-dual-core-setup` agent |
| Sensor / motor / bare-metal | Empty app | Add peripherals via Device Configurator and `mtb-add-library` agent |
| Graphics + connectivity | LVGL demo app, then add connectivity | Graphics is the hardest subsystem to bootstrap; connectivity is additive |

**To list available templates:**
```bash
project-creator-cli --list-apps
```

**To list available boards:**
```bash
project-creator-cli --list-boards
```

### For Existing Projects (Adding Capabilities)

If your project was already created with `project-creator-cli`, add capabilities incrementally using the available MTB agents:

1. **Add libraries** → `mtb-add-library` agent (creates `.mtb` files, updates Makefile)
2. **Add dual-core** → `mtb-dual-core-setup` agent (CM55 activation, IPC, boot sync)
3. **Add WiFi/MQTT** → `mtb-wifi-mqtt` agent (connection manager, MQTT client)
4. **Add LVGL graphics** → `mtb-lvgl-setup` agent (GFXSS personality, display drivers, VG-Lite)
5. **Add HTTP client** → `mtb-http-client` agent (REST API, JSON parsing)

---

## Building Up From Empty — The Goal

The practical advice today is: **start from the template closest to your goal**. For graphics, that means the LVGL demo. For WiFi, the WiFi template.

But the strategic goal is different: **it should always be possible to start from an empty app and build up to any project** by composing the right agents:

```
Empty App (via project-creator-cli)
  + mtb-dual-core-setup     → Active CM55
  + mtb-add-library         → Any middleware
  + mtb-wifi-mqtt           → WiFi + MQTT
  + mtb-lvgl-setup          → Graphics / touch display
  + mtb-ipc-patterns        → Cross-core data sharing
```

---

## What Goes Wrong Without `project-creator-cli`

### Anti-pattern 1: Clone and manually configure
```bash
# ❌ NEVER DO THIS
git clone https://github.com/Infineon/TARGET_KIT_PSE84_EVAL_EPC2
```
**Result:** Missing `design.modus` personalities, no generated code, `$(SEARCH_*)` variables undefined.

### Anti-pattern 2: Copy from a "working project" on GitHub
**Result:** API mismatches, stale BSP, cached build artifacts that mask dependency issues.

### Anti-pattern 3: `make getlibs` without Project Creator import
```bash
# ❌ INCOMPLETE
git clone <ce-template-repo>
make getlibs && make build   # FAILS
```
**Result:** CE templates are designed for Project Creator import, not standalone `make getlibs`.

---

## For AI Agents

1. **Never create a project by writing Makefiles and source files from scratch.**
2. **Never copy BSP files, `design.modus`, or `cycfg_*.c/h` from another project.**
3. **Never assume a "working example" found on GitHub is a valid reference.** Verify it is an official Infineon code example at the current SDK version.
4. **When a build fails with undefined symbols**, check whether the project was created via `project-creator-cli` before debugging further.
5. **When composing capabilities**, use the MTB agents in this workspace.
