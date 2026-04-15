---
name: mtb-help
description: ModusToolbox skill catalog and getting started guide. Use when the user asks what skills are available, what help exists, how to get started with ModusToolbox development, or asks for an overview of available capabilities.
---

# ModusToolbox™ Skill Catalog

> This workspace includes specialized skills for common ModusToolbox™ development tasks. Each skill provides instructions that Copilot loads automatically when relevant, or you can invoke manually with `/skill-name`.

---

## Getting Started

1. **Copy this template** into your ModusToolbox™ project (see README.md for instructions)
2. **Fill in `CONTEXT.md`** with your target device, kit, and software versions
3. **Start working** — Copilot will automatically use the right skill based on your task
4. **Or invoke directly** — type `/skill-name` to explicitly use a specific skill

---

## Available Skills by Category

### 🚀 Project Setup
| Skill | When it activates | What it does |
|-------|-------------------|--------------|
| `/project-creation` | **New project creation, build infrastructure issues** | Non-negotiable workflow rule: use `project-creator-cli`, choose the right template, build up capabilities via skills |
| `/dual-core-setup` | Starting a PSOC Edge project with active CM55 | Configures 3-project structure with boot handshake, IPC init, and CM55 application scaffolding |
| `/new-module` | Adding a new source file pair | Scaffolds `.c`/`.h` pair with correct file headers, include guards, and Doxygen stubs |

### 📡 Connectivity
| Skill | When it activates | What it does |
|-------|-------------------|--------------|
| `/wifi-mqtt` | Adding WiFi + MQTT to your project | WiFi STA connection, MQTT publish/subscribe, TLS configuration, reconnection logic |
| `/ble-setup` | Adding Bluetooth LE functionality | BTSTACK v4 API patterns, GATT service definition, BLE scanning, advertising |

### 🔧 Patterns & Fixes
| Skill | When it activates | What it does |
|-------|-------------------|--------------|
| `/ipc-patterns` | Dual-core communication beyond boot sync | Semaphore-guarded shared memory, ring buffer IPC, message queue patterns |
| `/retarget-io-fix` | Printf not working on PSOC Edge | Validated retarget-io init wrapper — `cy_retarget_io_init()` alone is insufficient on PSOC Edge |
| `/radar-dsp` | Building a radar signal processing pipeline | Range FFT, Doppler FFT, MTI filter, CM55 Helium/MVE acceleration patterns |

### 🛠️ Build & Configuration
| Skill | When it activates | What it does |
|-------|-------------------|--------------|
| `/build-error` | Build fails and you're stuck | Diagnoses common Makefile, linker, config errors with root-cause lookup |
| `/add-library` | Adding a middleware library | Library `.mtb` entry format, required COMPONENTS and DEFINES, dependency chains |
| `/device-configurator-spec` | Need to document peripheral config | Generates Device Configurator specification for MTB IDE setup |

### 📄 Documentation
| Skill | When it activates | What it does |
|-------|-------------------|--------------|
| `/readme` | Project needs a README | Generates a complete project README from code analysis + CONTEXT.md |

---

## Skills vs Custom Instructions

- **Skills** (this catalog) provide **task-specific** instructions loaded on demand
- **Custom instructions** (`CONTEXT.md`, `.github/copilot-instructions.md`) provide **always-on** context about your project

Both work together: custom instructions tell Copilot about your project; skills tell Copilot how to perform specific tasks.

---

## Managing Skills

```bash
/skills list          # List all available skills
/skills info <name>   # Details about a specific skill
/skills reload        # Reload after adding new skills
```

---

## Reference Documents

The `reference/` directory contains deep-dive guides that are NOT auto-loaded (zero token cost). Reference them explicitly when needed:

| Document | Contents |
|----------|----------|
| `reference/project-structures.md` | Project layouts for all device families (C3, PSOC 6, PSOC 4, XMC) |
| `reference/psoc-edge-dual-core-guide.md` | Dual-core architecture, IPC deep dive, boot sequence, debugging |
| `reference/psoc-edge-porting-guide.md` | Porting PSOC 6 examples to PSOC Edge, API migration, gotchas |
| `reference/middleware-compatibility.md` | Library versions, super-dependencies, compatibility matrix |
| `reference/build-system-patterns.md` | Makefile patterns, COMPONENTS, DEFINES, common.mk configuration |

**To use:** Tell Copilot to read a specific reference doc, e.g.:
> "Read `reference/psoc-edge-dual-core-guide.md` and help me set up IPC between CM33 and CM55"
