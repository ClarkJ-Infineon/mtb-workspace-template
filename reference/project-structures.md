# Project Structure Reference — All Device Families

This document contains the standard project layouts for ModusToolbox™ device families other than PSOC Edge (which is in `copilot-instructions.md` as the most complex/common case).

---

## PSOC Control C3 — Single-project layout

```
[project-name]/
├── .cyignore
├── .gitignore
├── LICENSE
├── Makefile                ← MTB_TYPE=APPLICATION (no MTB_PROJECTS needed)
├── README.md
├── common.mk               ← TARGET=KIT_T2G-C-M33-LITE_EVK (or similar)
├── main.c
├── source/
├── include/
└── deps/
    └── [library].mtb
```

**Notes:**
- Single Cortex-M33 core, optional TrustZone
- Uses `mtb_hal_` API (separate HAL library: `mtb-hal-psc3`)
- CAT1C PDL — same header (`cy_pdl.h`) as PSOC Edge but different device-specific peripherals

---

## PSOC 6 — Single-project or dual-CPU layout

**Single-project (most examples):**
```
[project-name]/
├── .cyignore
├── .gitignore
├── LICENSE
├── Makefile                ← MTB_TYPE=APPLICATION, TARGET=CY8CKIT-062S2-43012 (or similar)
├── README.md
├── main.c
├── source/
├── include/
└── deps/
    └── [library].mtb
```

**Dual-CPU projects** split into `proj_cm4/` and `proj_cm0p/` sub-projects when CM0+ runs independent code. Most PSOC 6 examples use single-project with the CM0+ running a pre-built sleep image from the BSP.

**Notes:**
- Uses `cyhal_` API (`mtb-hal-cat1`)
- CAT1 PDL
- WiFi/BT available on kits with CYW43xxx combo chips (add `DEFINES+=CYBSP_WIFI_CAPABLE`)
- Extensive library ecosystem — most MTB middleware was originally built for PSOC 6

---

## PSOC 4 — Single-project layout

```
[project-name]/
├── .cyignore
├── .gitignore
├── LICENSE
├── Makefile                ← TARGET=CY8CKIT-041S-MAX (or similar)
├── README.md
├── main.c
├── source/
├── include/
└── deps/
    └── [library].mtb
```

**PSOC 4 constraints:**
- Cortex-M0+ only — no FPU, no SIMD
- **CAT2 PDL** — same `Cy_` prefix but different header set from CAT1. Do not mix CAT1 and CAT2 code.
- MTB HAL (`cyhal_`) support is partial — verify function availability for CAT2 before using
- No WiFi/BT on-chip; AIROC combo chips require external connection
- Limited memory (typically 32-256 KB Flash, 4-32 KB SRAM)

---

## XMC1000 / XMC4000 — Single-project layout

```
[project-name]/
├── .cyignore
├── .gitignore
├── LICENSE
├── Makefile                ← TARGET=KIT_XMC47_RELAX_V1 (or similar)
├── README.md
├── main.c
├── source/
├── include/
└── deps/
    └── [library].mtb
```

**XMC API differences — critical:**
- Primary library is `mtb-xmclib-cat3` (XMCLib, `XMC_` prefix, e.g., `XMC_GPIO_SetOutputHigh()`)
- **Neither `cyhal_` nor `mtb_hal_` is available** — use XMCLib directly
- retarget-io IS available for XMC
- Device Configurator works but uses XMC-specific peripheral names
- FreeRTOS is available and commonly used

---

## XMC7000 — Single-project layout

```
[project-name]/
├── .cyignore
├── .gitignore
├── LICENSE
├── Makefile                ← TARGET=KIT_XMC72_EVK (or similar)
├── README.md
├── main.c
├── source/
├── include/
└── deps/
    └── [library].mtb
```

**Notes:**
- Uses `cyhal_` API with `mtb-hal-cat1` — **same stack as PSOC 6**, NOT XMCLib
- CAT1 PDL (`cy_pdl.h`)
- Cortex-M7 core with hardware FPU
- WiFi/BT available on some kits

---

## Library Dependency Files (.mtb format)

Each `.mtb` file in `deps/` contains a single line:
```
mtb://[library-name]#[version-tag]#$$ASSET_REPO$$/[library-name]/[version-tag]
```

Developer runs `make getlibs` after cloning to download all referenced libraries into `libs/`.

See the `#add-library` prompt for the full library table and COMPONENTS/DEFINES configuration.
