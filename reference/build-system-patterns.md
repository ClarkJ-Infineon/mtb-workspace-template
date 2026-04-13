# Build System Patterns — ModusToolbox Makefile Reference

Common Makefile patterns, COMPONENTS, DEFINES, and build configuration for ModusToolbox™ projects.

> **Source of truth:** Always check the library's GitHub README for authoritative configuration. This document captures patterns validated across multiple projects.

---

## Makefile Structure

### Single-project (PSOC 6, PSOC 4, XMC, PSOC C3)

```makefile
# Makefile (root)
MTB_TYPE=APPLICATION
APP_NAME=my-project

# Target and toolchain
TARGET=CY8CKIT-062S2-43012
TOOLCHAIN=GCC_ARM

# Source and include paths
SOURCES=$(wildcard source/*.c)
INCLUDES=include

# Libraries
COMPONENTS+=FREERTOS
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF

include $(CY_TOOLS_DIR)/make/start.mk
```

### 3-project (PSOC Edge)

**Root Makefile:**
```makefile
MTB_TYPE=APPLICATION
MTB_PROJECTS=proj_cm33_s proj_cm33_ns proj_cm55
include $(CY_TOOLS_DIR)/make/start.mk
```

**common.mk (shared settings):**
```makefile
TARGET=KIT_PSE84_EVAL_EPC2
TOOLCHAIN=GCC_ARM
```

**proj_cm33_ns/Makefile (application):**
```makefile
MTB_TYPE=PROJECT
APP_NAME=proj_cm33_ns

SOURCES=$(wildcard source/*.c)
INCLUDES=include

COMPONENTS+=FREERTOS LWIP MBEDTLS
DEFINES+=CYBSP_WIFI_CAPABLE
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF

# PSOC Edge printf fix
LDFLAGS+=-Wl,--wrap=_write

include ../common.mk
include $(CY_TOOLS_DIR)/make/start.mk
```

---

## COMPONENTS Reference

COMPONENTS enables pre-built library features. Add to the app Makefile's `COMPONENTS` variable.

| Component | What it enables | Common libraries |
|-----------|----------------|-----------------|
| `FREERTOS` | FreeRTOS kernel | freertos |
| `LWIP` | lwIP TCP/IP stack | lwip |
| `MBEDTLS` | Mbed TLS crypto | mbedtls |
| `BLUETOOTH` | BLE/BR-EDR stack | btstack-integration |

### How COMPONENTS works

ModusToolbox uses COMPONENTS to conditionally include source files. Inside library directories:

```
libs/freertos/
├── COMPONENT_FREERTOS/    ← Only compiled when COMPONENTS+=FREERTOS
│   └── ...
└── ...
```

If you forget `COMPONENTS+=FREERTOS`, the FreeRTOS source files are skipped and you get linker errors on FreeRTOS symbols.

---

## DEFINES Reference

DEFINES sets C preprocessor macros. Add to the app Makefile's `DEFINES` variable.

```makefile
# Common DEFINES patterns
DEFINES+=CYBSP_WIFI_CAPABLE                    # Enable WiFi on combo-chip kits
DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF    # Fix newlines for Windows terminals
DEFINES+=configTOTAL_HEAP_SIZE=0x20000          # FreeRTOS heap: 128 KB
DEFINES+=CY_MQTT_ACK_RECEIVE_TIMEOUT_MS=5000    # MQTT timeout override
```

### Precedence
- DEFINES in Makefile override library defaults
- common.mk DEFINES apply to all sub-projects (PSOC Edge)
- Sub-project Makefile DEFINES override common.mk for that sub-project only

---

## Source File Organization

### Recommended layout
```
proj_cm33_ns/
├── main.c                    ← Entry point, task creation
├── source/
│   ├── wifi_task.c           ← WiFi connection + monitoring
│   ├── mqtt_task.c           ← MQTT publish/subscribe
│   ├── sensor_task.c         ← Sensor data acquisition
│   └── app_bt_management.c   ← BLE callbacks (if using BLE)
├── include/
│   ├── app_config.h          ← All #defines (credentials, thresholds, pins)
│   ├── wifi_task.h
│   ├── mqtt_task.h
│   └── sensor_task.h
└── deps/
    ├── freertos.mtb
    ├── retarget-io.mtb
    └── wifi-connection-manager.mtb
```

### Configuration centralization
Keep all tunable values in `app_config.h`:
```c
/* app_config.h */
#define WIFI_SSID           "YOUR_SSID"
#define WIFI_PASSWORD       "YOUR_PASSWORD"
#define MQTT_BROKER_URL     "mqtt.example.com"
#define MQTT_BROKER_PORT    8883
#define SENSOR_READ_INTERVAL_MS  1000
```

---

## Build Commands

```bash
# Full build cycle
make getlibs          # Download dependencies (run after clone or adding .mtb)
make build            # Compile all
make program          # Flash to target

# Clean builds
make clean            # Remove build artifacts
make getlibs          # Re-download (if libs/ corrupted)

# Toolchain selection
make build TOOLCHAIN=GCC_ARM    # Default
make build TOOLCHAIN=ARM        # ARM Compiler 6
make build TOOLCHAIN=LLVM       # LLVM/Clang

# Verbose output (for debugging build issues)
make build VERBOSE=1

# Parallel build (faster on multi-core machines)
make build -j8
```

---

## Common Build Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `undefined reference to xTaskCreate` | Missing FREERTOS component | `COMPONENTS+=FREERTOS` |
| `undefined reference to cy_wcm_init` | Missing WiFi library or component | Add wifi-connection-manager .mtb + `DEFINES+=CYBSP_WIFI_CAPABLE` |
| `undefined reference to wiced_bt_stack_init` | Missing BLUETOOTH component | `COMPONENTS+=BLUETOOTH` |
| `fatal error: cy_pdl.h: No such file` | BSP not installed | Run `make getlibs` |
| `No rule to make target 'build'` | Wrong directory | Run from project root (where Makefile is) |
| `TARGET not found` | Wrong kit name | Check CONTEXT.md for correct TARGET value |
| `multiple definition of main` | App code in wrong sub-project | All app code in `proj_cm33_ns/` only |

---

## .cyignore

Controls which files the MTB IDE indexes. Place in project root:

```
# .cyignore — exclude from MTB IDE indexing
libs/btstack/docs
libs/*/test
libs/*/TESTS
```

This speeds up IDE loading but doesn't affect the build.

---

## .gitignore

Standard ModusToolbox .gitignore:

```gitignore
# Build output
build/
libs/
GeneratedSource/

# IDE files
.vscode/
.metadata/
*.launch

# OS files
.DS_Store
Thumbs.db

# MTB shared libraries (downloaded by make getlibs)
mtb_shared/
```

**Critical:** `libs/` and `build/` must be gitignored. Library source is downloaded by `make getlibs` from `.mtb` file URLs. Committing `libs/` bloats the repo and causes version conflicts.
