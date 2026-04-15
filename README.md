# MTB Workspace Copilot Template

A GitHub Copilot context template for [ModusToolbox™](https://www.infineon.com/modustoolbox) application development. Drop these files into any ModusToolbox project to give GitHub Copilot detailed knowledge of Infineon device families, project structure, APIs, and development conventions.

Works with **GitHub Copilot Chat in VS Code**, **GitHub Copilot Workspace**, and **GitHub Copilot CLI**.

**Supported device families:** PSOC Edge · PSOC Control C3 · PSOC 6 · PSOC 4 · XMC1000/4000 · XMC7000

### Copilot Customization Formats

| Format | Location | Supported Tools | Invocation |
|--------|----------|-----------------|------------|
| **Custom Instructions** | `.github/copilot-instructions.md` | All (auto-loaded) | Automatic — loaded every session |
| **Skills** ([open standard](https://agentskills.io)) | `.github/skills/*/SKILL.md` | VS Code Chat · Copilot CLI · Cloud Agent | Type `/skill-name` in chat |
| **Agents** | `.github/agents/*.agent.md` | VS Code Chat · Copilot CLI · Cloud Agent | Agent picker dropdown or auto-loaded |

Skills are the primary invocable content — each skill is a self-contained knowledge module that loads on-demand when relevant. Agents provide broader persistent personas for autonomous multi-step tasks.

---

## Prerequisites

- [ModusToolbox™](https://www.infineon.com/modustoolbox) installed
- [Visual Studio Code](https://code.visualstudio.com/) with the [GitHub Copilot extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)
- An active [GitHub Copilot subscription](https://github.com/features/copilot/plans)

---

## Installation

> **No scripts to run.** Copy selected items from this repo into your MTB project root.

### Step 1 — Download this repository

Click **Code → Download ZIP** on this page, then extract it.

*Or clone it:*
```
git clone https://github.com/ClarkJ-Infineon/mtb-workspace-template.git
```

### Step 2 — Copy files into your MTB project

From the extracted/cloned folder, copy these items into the **root of your ModusToolbox project**:

```
.github/          → your-mtb-project/.github/
CONTEXT.md        → your-mtb-project/CONTEXT.md
reference/        → your-mtb-project/reference/
memories/         → your-mtb-project/memories/
```

The `.github/` folder includes:
- `copilot-instructions.md` — system instructions (loaded automatically every session)
- `skills/` — skills for VS Code Chat, Copilot CLI, and Cloud Agent (invoked via `/skill-name`)
- `agents/` — capability agents for autonomous multi-step tasks

> **Do NOT copy** `README.md`, `README-TEMPLATE.md`, or `LICENSE` — these are about the template itself, not your project.
>
> **For your project README:** Copy `README-TEMPLATE.md` to your project root, rename it to `README.md`, and fill in the sections. Or use `/readme` in Copilot Chat to generate one from your code.

> **If your project already has a `.github/` folder** (e.g., with GitHub Actions workflows), copy the contents rather than replacing the folder:
> - Copy `.github/copilot-instructions.md` into your existing `.github/`
> - Copy the `.github/skills/` subfolder into your existing `.github/`
> - Copy the `.github/agents/` subfolder into your existing `.github/`

### Step 3 — Fill in CONTEXT.md

Open `CONTEXT.md` in your MTB project and fill in your target device:

```markdown
| **Device family** | PSOC Edge       |  ← replace with your device
| **Evaluation kit** | KIT_PSE84_EVAL_EPC2 |
| **MTB TARGET value** | KIT_PSE84_EVAL_EPC2 |
```

Update the software versions table if your versions differ from the defaults.

That's it. Open the project in VS Code — Copilot now has full MTB context.

---

## Using with VS Code Copilot Chat

### Every session — attach CONTEXT.md first

`copilot-instructions.md` loads automatically. `CONTEXT.md` does not — attach it at the start of each session:

```
#file:CONTEXT.md  [your question here]
```

**Example first messages:**
```
#file:CONTEXT.md  Help me set up a FreeRTOS task for UART logging.

#file:CONTEXT.md  What project structure should I use for this PSOC Edge application?

#file:CONTEXT.md  /build-error  [paste your build output here]
```

### Available skills

Type `/` in Copilot Chat to see available skills. Use `/mtb-help` for the full catalog with descriptions.

**Discovery:**
| Skill | What it does |
|-------|-------------|
| `/mtb-help` | Full catalog of all available skills with descriptions and usage examples |

**Project Setup:**
| Skill | What it does |
|-------|-------------|
| `/dual-core-setup` | Configure PSOC Edge for active CM55: boot handshake, IPC init, project structure |
| `/new-module` | Scaffold a new `.c`/`.h` source module with correct Doxygen file headers |

**Connectivity:**
| Skill | What it does |
|-------|-------------|
| `/wifi-mqtt` | WiFi STA connection, MQTT publish/subscribe, TLS configuration, reconnection |
| `/http-client` | HTTP GET via cy_secure_sockets, JSON parsing with coreJSON, polling patterns |
| `/ble-setup` | BTSTACK v4 API patterns, GATT service definition, BLE scanning |

**Patterns & Fixes:**
| Skill | What it does |
|-------|-------------|
| `/ipc-patterns` | Semaphore guard, shared memory, ring buffer IPC for dual-core |
| `/transparent-printf` | Cross-core printf via shared-memory ring buffer and `--wrap=_write` |
| `/retarget-io-fix` | Fix printf not working on PSOC Edge (retarget-io init wrapper) |
| `/radar-dsp` | Radar FFT pipeline, MTI filter, CM55 Helium/MVE SIMD acceleration |
| `/lvgl-setup` | LVGL v9 graphics with VG-Lite GPU, GFXSS personality, display drivers |

**Build & Configuration:**
| Skill | What it does |
|-------|-------------|
| `/build-error` | Diagnose build failures — common MTB root causes and fixes |
| `/add-library` | Add a library dependency with correct .mtb entry, COMPONENTS, DEFINES |
| `/device-configurator-spec` | Generate a Device Configurator setup specification |
| `/openocd-debug` | OpenOCD/GDB debugging, multi-core debug, CFSR fault analysis |

**Documentation:**
| Skill | What it does |
|-------|-------------|
| `/readme` | Generate a project README from code analysis + CONTEXT.md |

> Skills are also auto-loaded by Copilot based on relevance — you don't always need to invoke them explicitly.

---

## Using with Copilot CLI

If you use [GitHub Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli) (terminal-based), the template includes **5 consolidated agents** in `.github/agents/`. These are broader than individual skills — each agent covers a full capability domain and can handle multi-step tasks autonomously. Skills are also available to CLI users.

| Agent | Scope |
|-------|-------|
| `mtb-project` | Project creation via `project-creator-cli`, library dependency management (`.mtb` files), Makefile configuration |
| `mtb-connectivity` | WiFi STA, MQTT pub/sub, HTTP client, TLS, JSON parsing, reconnection strategies |
| `mtb-multicore` | CM55 activation, boot synchronization, IPC (shared memory, message queue, ring buffer), cross-core printf |
| `mtb-display` | LVGL v9 graphics, VG-Lite GPU acceleration, GFXSS personality, display/touch drivers |
| `mtb-diagnostics` | Build error diagnosis, printf fix, OpenOCD/GDB debugging, CFSR fault analysis |

Agents are loaded automatically when relevant to your task. The `copilot-instructions.md` file provides shared context for both VS Code and CLI sessions.

---

## What Copilot Knows

Once installed, Copilot understands:

| Topic | Details |
|-------|---------|
| **Project structure** | 3-project layout for PSOC Edge (cm33_s / cm33_ns / cm55); single-project for all other families |
| **HAL per device** | `mtb_hal_` for PSOC Edge and Control; `cyhal_` for PSOC 6, PSOC 4, XMC7000; `XMC_` (XMCLib) for XMC1000/4000 |
| **Dual-core patterns** | Boot sync, IPC message queue, shared memory, ring buffer, retarget-io fix |
| **Connectivity** | WiFi STA, MQTT publish/subscribe, HTTP client, TLS, BLE GATT server, BLE scanning |
| **Radar DSP** | Range FFT, Doppler FFT, MTI filter, CM55 Helium/MVE acceleration |
| **Graphics** | LVGL v9 with VG-Lite GPU, GFXSS personality, display/touch drivers, framebuffer management |
| **Debugging** | OpenOCD/GDB multi-core debug, CFSR fault register analysis, PPU violations |
| **Library dependencies** | `deps/*.mtb` format, dependency chains, COMPONENTS/DEFINES configuration |
| **Build system** | `make build`, `make getlibs`, `make program`; TOOLCHAIN options (GCC_ARM, ARM, LLVM) |
| **Code standards** | Doxygen file headers, include guards, error handling patterns |
| **Reference docs** | Deep-dive guides in `reference/` — porting, middleware compatibility, build patterns |

---

## CONTEXT.md Reference

`CONTEXT.md` is the single source of truth for your project. Keep it current — stale context leads to stale suggestions.

**Most important fields:**

| Field | Why it matters |
|-------|---------------|
| `Device family` | Drives project structure, API selection, HAL repo |
| `MTB TARGET value` | Used in Makefile (`TARGET=`) and `make program` |
| `ModusToolbox™ version` | Prevents version drift in generated code |
| `Project description` | Gives Copilot purpose and constraint context |

**Update CONTEXT.md when:**
- ModusToolbox ships a new version
- You change hardware target
- You add new constraints or rules

---

## Customizing for Your Project

Add project-specific rules at the bottom of `.github/copilot-instructions.md`:

```markdown
## Project-Specific Rules

- No dynamic memory allocation — use fixed pools only
- Build with ARM Compiler 6 only: TOOLCHAIN=ARM
- All timing uses DWT->CYCCNT
```

These rules are applied every session on top of the standard MTB rules.

---

## Committing to Your Project

The template files are designed to be committed alongside your project code:

```
git add .github/ CONTEXT.md reference/ memories/
git commit -m "Add MTB Copilot workspace template"
```

The `memories/session/current.md` file is a session state placeholder — update it as you work if using session continuity features.

---

## Reference Links

| Resource | URL |
|----------|-----|
| **Copilot Customization Overview** | https://code.visualstudio.com/docs/copilot/copilot-customization |
| **Agent Skills (open standard)** | https://code.visualstudio.com/docs/copilot/customization/agent-skills |
| **Custom Agents** | https://code.visualstudio.com/docs/copilot/customization/custom-agents |
| **Prompt Files** | https://code.visualstudio.com/docs/copilot/customization/prompt-files |
| **Agent Skills Specification** | https://agentskills.io |
| **Community Examples** | https://github.com/github/awesome-copilot |
| ModusToolbox™ library index (all device families) | https://github.com/Infineon/modustoolbox-software/blob/master/README.md |
| MTB PDL API Reference (CAT1/CAT1C — PSOC 6, Edge, Control) | https://infineon.github.io/mtb-pdl-cat1/pdl_api_reference_manual/html/ |
| MTB PDL API Reference (CAT2 — PSOC 4) | https://infineon.github.io/mtb-pdl-cat2/pdl_api_reference_manual/html/ |
| MTB HAL API Reference (cyhal_ — PSOC 6/4/XMC7000) | https://infineon.github.io/mtb-hal-cat1/html/modules.html |
| PSOC Control C3 HAL (mtb_hal_) | https://github.com/Infineon/mtb-hal-psc3 |
| PSOC Edge DSL (PSE84 GO) | https://github.com/Infineon/mtb-dsl-pse8xxgo |
| PSOC Edge DSL (PSE84 GP) | https://github.com/Infineon/mtb-dsl-pse8xxgp |
| XMCLib (XMC1000/4000) | https://github.com/Infineon/mtb-xmclib-cat3 |
| Infineon MTB Code Examples | https://github.com/Infineon/?q=mtb-example |
| ModusToolbox Documentation | https://www.infineon.com/modustoolbox |

---

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
