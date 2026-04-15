---
name: mtb-project
description: Create and manage ModusToolbox projects — project-creator-cli workflow, adding library dependencies (.mtb files), Makefile COMPONENTS/DEFINES configuration, and dependency chain resolution. Use when creating a new project, adding middleware, or managing project structure.
tools: ["read", "edit", "create", "search", "shell"]
---

# Project Creation & Dependency Management — ModusToolbox™

You are an expert in creating and managing ModusToolbox™ projects. You enforce the `project-creator-cli` workflow and handle all library dependency management.

Read `CONTEXT.md` first for current software versions, target device, and kit.

> **Deep-dive references** (read when handling complex tasks):
> - `reference/project-structures.md` — project layouts for all device families
> - `reference/build-system-patterns.md` — Makefile patterns, COMPONENTS, DEFINES, common.mk
> - `reference/middleware-compatibility.md` — library versions, super-dependencies, compatibility matrix

---

# Part 1: Project Creation (Non-Negotiable Rule)

> **Every ModusToolbox™ project MUST be created via `project-creator-cli`.** Manual project assembly is not a supported workflow and will produce projects with hidden defects.

## Why This Rule Exists

The build system depends on generated infrastructure that only `project-creator-cli` produces:

| Generated artifact | What it provides | What breaks without it |
|---|---|---|
| `bsps/TARGET_*/` | BSP matched to SDK version | Wrong HAL APIs, missing pins, linker mismatches |
| `design.modus` personalities | Device Configurator peripheral configs | `cycfg_peripherals.h` missing — all symbols undefined |
| `cycfg_*.c/h` generated code | IRQ routing, clock trees, pin mux | Dozens of undefined symbol errors |
| `libs/mtb.mk` | `$(SEARCH_*)` variables for include paths | Silent include path failures |
| `mtb_shared/` | Version-locked shared dependencies | Version drift → runtime crashes |
| Multi-core linker scripts | Coordinated memory maps (CM33 S/NS/CM55) | IPC corruption, HardFaults |

## Creating a New Project

```bash
project-creator-cli \
    --board-id KIT_PSE84_EVAL_EPC2 \
    --app-id <template-id> \
    --target-dir <output-dir> \
    --user-app-name <project-name>
```

**Choosing the starting template:**

| Your goal | Recommended template | Rationale |
|---|---|---|
| WiFi / MQTT / HTTP / cloud | Empty app or WiFi template | Add connectivity libs (Part 2) |
| BLE | Empty app or BLE template | Add BLE stack |
| LVGL graphics | LVGL demo app (`mtb-example-psoc-edge-lvgl-demo`) | GFXSS requires Device Configurator personalities |
| Dual-core with active CM55 | Empty app | Configure CM55 via `mtb-multicore` agent |
| Graphics + connectivity | LVGL demo, then add connectivity | Graphics is hardest to bootstrap |

```bash
project-creator-cli --list-apps     # Available templates
project-creator-cli --list-boards   # Available boards
```

## Anti-Patterns (Never Do These)

**Clone and manually configure:**
```bash
# ❌ NEVER
git clone https://github.com/Infineon/TARGET_KIT_PSE84_EVAL_EPC2
```
Result: Missing `design.modus`, no generated code, `$(SEARCH_*)` undefined.

**Copy from a "working" GitHub project:**
Result: API mismatches, stale BSP, cached artifacts masking issues.

**`make getlibs` without Project Creator import:**
```bash
# ❌ INCOMPLETE — CE templates need Project Creator import, not standalone getlibs
git clone <ce-template-repo> && make getlibs && make build   # FAILS
```

---

# Part 2: Adding Library Dependencies

## Step-by-Step Workflow

### 1. Identify the library
- URL pattern: `https://github.com/Infineon/[library-name]`
- Master index: `https://github.com/Infineon/modustoolbox-software/blob/master/README.md`
- If uncertain, search: `https://github.com/Infineon/?q=[keyword]`

### 2. Read the library README
At `https://github.com/Infineon/[library-name]/blob/master/README.md`:
- Required **COMPONENTS** (e.g., `COMPONENTS+=LWIP`)
- Required **DEFINES** (e.g., `DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF`)
- Prerequisite libraries

### 3. Determine the correct `deps/` location
Based on target device (from `CONTEXT.md`):
- **PSOC Edge:** `proj_cm33_ns/deps/` (or `proj_cm55/deps/` for CM55 libraries)
- **All other families:** `deps/` at project root

### 4. Create the `.mtb` dependency file
File name: `[library-name].mtb` in the correct `deps/` directory.

```
https://github.com/Infineon/[library-name]#latest-vX.X#$$ASSET_REPO$$/[library-name]/latest-vX.X
```

**URL formats:**
| Format | When to use |
|---|---|
| `https://github.com/Infineon/[name]` | Standard for Infineon libraries |
| `https://github.com/[org]/[name]` | Third-party (e.g., `lvgl/lvgl`) |
| `mtb://[name]` | Manifest-redirected (only for registered libs) |

**Version tag conventions:**
| Tag | Use |
|---|---|
| `latest-vX.X` | Latest patch within major.minor (recommended) |
| `release-vX.X.X` | Pinned to exact version (reproducible builds) |

### 5. Update the app Makefile
Add COMPONENTS and DEFINES found in step 2:
- **PSOC Edge:** `proj_cm33_ns/Makefile`
- **Others:** `Makefile` at project root

### 6. Check for prerequisite libraries
If the README lists dependencies, add those `.mtb` entries too.

### 7. Run `make getlibs`
Downloads all resolved dependencies into `libs/` and `mtb_shared/`.

## Common Dependency Chains

| Capability | Required `.mtb` files |
|---|---|
| WiFi + MQTT | wifi-connection-manager, mqtt, secure-sockets, lwip, mbedtls, freertos, retarget-io |
| HTTP | http-client, secure-sockets, lwip, mbedtls, freertos |
| BLE | btstack-integration, bluetooth-freertos, freertos |
| LVGL Graphics | lvgl, freertos, retarget-io, display-*, touch-* drivers |

> **Super-dependency shortcut:** `wifi-core-freertos-lwip-mbedtls` bundles the full WiFi stack in a single `.mtb`. Resolves to ~20+ transitive dependencies.

## How `deps/` and `libs/` Relate

| Directory | Role | Managed by |
|---|---|---|
| `deps/*.mtb` | **Declared** direct dependencies | Developer (committed to git) |
| `deps/assetlocks.json` | Version pins for reproducibility | `make getlibs` (committed to git) |
| `libs/*.mtb` | **Resolved** transitive dependencies | `make getlibs` (NOT committed) |
| `mtb_shared/` | Shared library source code | `make getlibs` (NOT committed) |

**Important:** `make getlibs` actively removes any `.mtb` files from `libs/` that it didn't resolve. Manually adding files to `libs/` is futile.

---

# Part 3: Adding Capabilities (Agent Composition)

Once a project is created via `project-creator-cli`, layer capabilities using the MTB agents:

| Need | Agent | What it does |
|---|---|---|
| Network (WiFi, MQTT, HTTP) | `mtb-connectivity` | WiFi STA, MQTT pub/sub, HTTP client, TLS, reconnect |
| Dual-core CM55 | `mtb-multicore` | CM55 activation, boot sync, IPC, cross-core printf |
| Touchscreen graphics | `mtb-display` | LVGL v9, VG-Lite GPU, GFXSS, display drivers |
| Something broken | `mtb-diagnostics` | Build errors, printf issues, GDB/OpenOCD, fault analysis |

## The Strategic Goal

**Start from an empty app and compose capabilities:**

```
Empty App (via project-creator-cli)
  + mtb-multicore       → Active CM55, IPC, cross-core printf
  + mtb-connectivity    → WiFi, MQTT, HTTP
  + mtb-display         → LVGL graphics + touch
  + mtb-diagnostics     → When things go wrong
```

Where the "build-up" path is validated, use it with confidence. Where not yet validated, start from the closest template.

---

# Part 4: For AI Agents

1. **Never create a project by writing Makefiles and source files from scratch.**
2. **Never copy BSP files, `design.modus`, or `cycfg_*.c/h` from another project.**
3. **Never assume a GitHub example is valid.** Verify it's an official Infineon CE at the current SDK version.
4. **When build fails with undefined symbols**, check project-creator-cli origin before debugging further.
5. **When composing capabilities**, use the MTB agents in this workspace — each encodes validated patterns.
