# Project Context — [Your Project Name]

> This file is the single source of truth for this workspace.
> Update it whenever software versions change, project scope shifts, or new constraints are added.
> Copilot reads this at every session start — stale information here leads to stale outputs.

---

## Current Software Versions

> **Always update this table first when a new release ships.**
> All instructions reference this table rather than hardcoding version numbers.

| Product | Version | Notes |
|---------|---------|-------|
| ModusToolbox™ | **3.7** | Update when new release ships |
| GCC ARM Toolchain | **14.2.1** | Bundled with MTB tools |
| ARM Compiler 6 | **6.22** | Available in MTB tools if licensed |
| BSP version | **check mtb-bsps repo** | Target-specific |

---

## Target Hardware

**Fill in ONE row below; delete the others.**

| Field | Value |
|-------|-------|
| **Device family** | PSOC Edge / PSOC Control C3 / PSOC 6 / PSOC 4 / XMC *(pick one)* |
| **Specific device** | e.g., PSE84 / PSOC C3 / PSOC 62 / PSOC 4100S Plus / XMC4800 |
| **Evaluation kit** | e.g., KIT_PSE84_EVAL_EPC2 |
| **MTB TARGET value** | e.g., `KIT_PSE84_EVAL_EPC2` — used in Makefile and `make program` |
| **Core architecture** | CM55+CM33 (Edge) / CM33 (C3) / CM4+CM0+ (PSOC 6) / CM0+ (PSOC 4) / CM4 or CM0+ (XMC) |
| **Project layout** | 3-project (PSOC Edge) / Single-project (all others) |
| **Primary API** | mtb_hal_ + Cy_ (PSOC Edge/Control) / cyhal_ + Cy_ (PSOC 6) / Cy_ limited (PSOC 4) / XMC_ (XMC) |
| **Special features** | e.g., Helium MVE (Edge CM55), TrustZone (Edge/Control), WiFi/BT (AIROC), motor control |

**Common kit TARGET values by family:**

| Family | Kit | TARGET |
|--------|-----|--------|
| PSOC Edge E84 | KIT_PSE84_EVAL_EPC2 | `KIT_PSE84_EVAL_EPC2` |
| PSOC Control C3 | KIT_T2G-C-M33-LITE_EVK | `KIT_T2G-C-M33-LITE_EVK` |
| PSOC 6 (WiFi/BT) | CY8CKIT-062S2-43012 | `CY8CKIT-062S2-43012` |
| PSOC 6 (BLE) | CY8CKIT-062-BLE | `CY8CKIT-062-BLE` |
| PSOC 4 | CY8CKIT-041S-MAX | `CY8CKIT-041S-MAX` |
| XMC4800 | KIT_XMC48_RELAX_ECAT_V1 | `KIT_XMC48_RELAX_ECAT_V1` |
| XMC1400 | KIT_XMC14_BOOT_001 | `KIT_XMC14_BOOT_001` |

---

## Project Description

**What this project does:**
[One paragraph — what the application does, what problem it solves]

**Key middleware / libraries used:**
- FreeRTOS — task scheduling
- [Add others as relevant; note XMC uses XMCLib (not cyhal_/mtb_hal_); PSOC Edge/Control use mtb_hal_, not cyhal_]

**Performance targets / constraints:**
- [e.g., decode latency < 100ms]
- [e.g., must fit in 256KB flash]

---

## Active Work

| Task | Status | Notes |
|------|--------|-------|
| [Task name] | In progress | [Brief note] |

---

## Non-Negotiable Rules for This Project

*(Add project-specific rules here — these apply IN ADDITION to the workspace rules in `.github/copilot-instructions.md`)*

- [e.g., "No dynamic memory allocation — fixed pools only"]
- [e.g., "All timing measurements use DWT->CYCCNT"]
- [e.g., "Build with ARM Compiler 6 only: TOOLCHAIN=ARM"]

---

## Reference Links

| Resource | URL |
|----------|-----|
| **MTB library master index (all families)** | https://github.com/Infineon/modustoolbox-software/blob/master/README.md |
| MTB PDL API Reference (CAT1/CAT1C — PSOC 6, Edge, Control) | https://infineon.github.io/mtb-pdl-cat1/pdl_api_reference_manual/html/ |
| MTB PDL API Reference (CAT2 — PSOC 4) | https://infineon.github.io/mtb-pdl-cat2/pdl_api_reference_manual/html/ |
| MTB HAL (cyhal_ — PSOC 6 / PSOC 4) | https://infineon.github.io/mtb-hal-cat1/html/modules.html |
| MTB HAL (mtb_hal_ — PSOC Control C3) | https://github.com/Infineon/mtb-hal-psc3 |
| MTB HAL / DSL (mtb_hal_ — PSOC Edge PSE84 GO) | https://github.com/Infineon/mtb-dsl-pse8xxgo |
| MTB HAL / DSL (mtb_hal_ — PSOC Edge PSE84 GP) | https://github.com/Infineon/mtb-dsl-pse8xxgp |
| PSOC 4 HAL (cyhal_ — CAT2) | https://github.com/Infineon/mtb-hal-cat2 |
| XMCLib (XMC1000/4000) | https://github.com/Infineon/mtb-xmclib-cat3 |
| XMC7000 uses mtb-hal-cat1 (cyhal_) — same as PSOC 6 | https://github.com/Infineon/mtb-hal-cat1 |
| Infineon MTB Examples (GitHub — all families) | https://github.com/Infineon/?q=mtb-example |
| Kit Guide (update URL for your kit) | https://www.infineon.com/cms/en/product/evaluation-boards/ |
| ModusToolbox Documentation | https://www.infineon.com/modustoolbox |
