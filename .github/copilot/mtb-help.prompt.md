# ModusToolbox Prompt Catalog — v2

> This template includes specialized pattern prompts for common ModusToolbox development tasks. Each prompt is a self-contained recipe that you invoke by typing `#prompt-name` in Copilot Chat.

---

## Getting Started

1. **Copy this template** into your ModusToolbox project (see README.md for instructions)
2. **Fill in `CONTEXT.md`** with your target device, kit, and software versions
3. **Attach `CONTEXT.md`** in your first Copilot Chat message each session
4. **Invoke any prompt below** by typing `#` followed by the prompt name

---

## Available Prompts by Category

### 🔍 Discovery
| Prompt | Description |
|--------|-------------|
| `#mtb-help` | You're looking at it! Full catalog of available prompts. |

### 🚀 Project Setup
| Prompt | When to use | What it does |
|--------|------------|--------------|
| `#dual-core-setup` | Starting a PSOC Edge project with active CM55 | Configures 3-project structure with boot handshake, IPC init, and CM55 application scaffolding |
| `#new-module` | Adding a new source file pair | Scaffolds `.c`/`.h` pair with correct file headers, include guards, and Doxygen stubs |

### 📡 Connectivity
| Prompt | When to use | What it does |
|--------|------------|--------------|
| `#wifi-mqtt` | Adding WiFi + MQTT to your project | WiFi STA connection, MQTT publish/subscribe, TLS configuration, reconnection logic |
| `#ble-setup` | Adding Bluetooth LE functionality | BTSTACK v4 API patterns, GATT service definition, BLE scanning, advertising |

### 🔧 Patterns & Fixes
| Prompt | When to use | What it does |
|--------|------------|--------------|
| `#ipc-patterns` | Dual-core communication beyond boot sync | Semaphore-guarded shared memory, ring buffer IPC, message queue patterns |
| `#retarget-io-fix` | Printf not working on PSOC Edge | Validated retarget-io init wrapper — `cy_retarget_io_init()` alone is insufficient on PSOC Edge |
| `#radar-dsp` | Building a radar signal processing pipeline | Range FFT, Doppler FFT, MTI filter, CM55 Helium/MVE acceleration patterns |

### 🛠️ Build & Configuration
| Prompt | When to use | What it does |
|--------|------------|--------------|
| `#build-error` | Build fails and you're stuck | Diagnoses common Makefile, linker, config errors with root-cause lookup |
| `#add-library` | Adding a middleware library | Library `.mtb` entry format, required COMPONENTS and DEFINES, dependency chains |
| `#device-configurator-spec` | Need to document peripheral config | Generates Device Configurator specification for MTB IDE setup |

### 📄 Documentation
| Prompt | When to use | What it does |
|--------|------------|--------------|
| `#readme` | Project needs a README | Generates a complete project README from code analysis + CONTEXT.md |

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

---

## Adding New Prompts (for template maintainers)

1. Create `.github/copilot/<name>.prompt.md` with the prompt content
2. Add one row to the pattern index table in `.github/copilot-instructions.md`
3. Add a full entry to this catalog (`#mtb-help`) under the appropriate category
4. Update the version number in the header above
