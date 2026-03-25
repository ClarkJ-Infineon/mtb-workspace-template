# MTB Workspace Copilot Template

A GitHub Copilot context template for [ModusToolbox™](https://www.infineon.com/modustoolbox) application development. Drop these files into any ModusToolbox project to give GitHub Copilot detailed knowledge of Infineon device families, project structure, APIs, and development conventions.

Works with **GitHub Copilot Chat in VS Code** and **GitHub Copilot CLI**.

**Supported device families:** PSOC Edge · PSOC Control C3 · PSOC 6 · PSOC 4 · XMC1000/4000 · XMC7000

---

## Prerequisites

- [ModusToolbox™](https://www.infineon.com/modustoolbox) installed
- [Visual Studio Code](https://code.visualstudio.com/) with the [GitHub Copilot extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)
- An active [GitHub Copilot subscription](https://github.com/features/copilot/plans)

---

## Installation

> **No scripts to run.** Copy three items from this repo into your MTB project root.

### Step 1 — Download this repository

Click **Code → Download ZIP** on this page, then extract it.

*Or clone it:*
```
git clone https://github.com/ClarkJ-Infineon/mtb-workspace-template.git
```

### Step 2 — Copy files into your MTB project

From the extracted/cloned folder, copy these three items into the **root of your ModusToolbox project**:

```
.github/          → your-mtb-project/.github/
CONTEXT.md        → your-mtb-project/CONTEXT.md
memories/         → your-mtb-project/memories/
```

> **If your project already has a `.github/` folder** (e.g., with GitHub Actions workflows), copy the contents rather than replacing the folder:
> - Copy `.github/copilot-instructions.md` into your existing `.github/`
> - Copy the `.github/copilot/` subfolder into your existing `.github/`

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

#file:CONTEXT.md  #build-error  [paste your build output here]
```

### Available skills

Type `#` in Copilot Chat to invoke the included prompt files:

| Skill | What it does |
|-------|-------------|
| `#add-library` | Add a library dependency — reads the library's GitHub README for correct COMPONENTS and DEFINES, creates the `.mtb` entry, updates the Makefile |
| `#new-module` | Scaffold a new `.c`/`.h` source module with correct Doxygen file headers populated from your CONTEXT.md |
| `#device-configurator-spec` | Generate a Device Configurator setup specification for a peripheral (what to configure in the MTB IDE GUI) |
| `#build-error` | Diagnose a build failure — works through common MTB root causes: missing `make getlibs`, wrong HAL prefix, missing COMPONENTS, wrong TARGET, and more |

> If `#` does not surface the prompt files in your VS Code version, open the relevant file from `.github/copilot/` and paste it into the chat window directly.

---

## What Copilot Knows

Once installed, Copilot understands:

| Topic | Details |
|-------|---------|
| **Project structure** | 3-project layout for PSOC Edge (cm33_s / cm33_ns / cm55); single-project for all other families |
| **HAL per device** | `mtb_hal_` for PSOC Edge and Control; `cyhal_` for PSOC 6, PSOC 4, XMC7000; `XMC_` (XMCLib) for XMC1000/4000 |
| **HAL repos** | Correct source repo per device — DSL repos for PSOC Edge, `mtb-hal-psc3` for Control, `mtb-hal-cat1/cat2` for PSOC 6/4 |
| **Library dependencies** | `deps/*.mtb` format, `make getlibs`, where to find COMPONENTS/DEFINES in each library's GitHub README |
| **Build system** | `make build`, `make getlibs`, `make program`; TOOLCHAIN options (GCC_ARM, ARM, LLVM) |
| **Device Configurator** | What cannot be generated automatically; how to specify peripheral configuration for the IDE GUI |
| **Code standards** | Doxygen file headers, include guards, error handling patterns |
| **Naming rules** | PSOC always ALL CAPS — never "PSoC" |

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
git add .github/ CONTEXT.md memories/
git commit -m "Add MTB Copilot workspace template"
```

The `memories/session/current.md` file is a session state placeholder — update it as you work if using session continuity features.

---

## Reference Links

| Resource | URL |
|----------|-----|
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
