# Middleware Compatibility Reference

Library versions, super-dependencies, and compatibility notes for ModusToolbox™ middleware.

> **Important:** Always check the library's GitHub README for the authoritative, current configuration. This document provides a quick reference but library versions update frequently.

---

## Library Dependency Chains

Many ModusToolbox libraries have "super-dependencies" — adding one library requires several others. Here are the common chains:

### WiFi + MQTT Chain
```
mqtt
├── aws-iot-device-sdk-port
├── secure-sockets
│   ├── mbedtls
│   └── lwip
├── wifi-connection-manager
│   ├── wifi-core-freertos-lwip-mbedtls
│   │   ├── lwip
│   │   ├── mbedtls
│   │   ├── freertos
│   │   └── wifi-host-driver
│   └── whd-bsp-integration
└── freertos
```

**Shortcut:** Adding `wifi-connection-manager` and `mqtt` to deps/ handles most of the chain — `make getlibs` resolves transitive dependencies.

### BLE Chain
```
btstack-integration
├── bluetooth-freertos
│   └── freertos
├── btstack
└── btsdk-ble-*(optional profile libraries)
```

### HTTP Client Chain
```
http-client
├── secure-sockets
│   ├── mbedtls
│   └── lwip
├── wifi-connection-manager (if using WiFi)
└── freertos
```

---

## .mtb File Format

Each `.mtb` file in `deps/` contains exactly one line:
```
mtb://[library-name]#[version-tag]#$$ASSET_REPO$$/[library-name]/[version-tag]
```

### Version Tag Conventions
- `latest-v3.X` — latest patch within major.minor (recommended for most)
- `release-v3.2.0` — pinned to exact version (for reproducible builds)
- `main` — bleeding edge (not recommended for production)

### Common Library .mtb Entries

```
# Core infrastructure
mtb://freertos#latest-v10.X#$$ASSET_REPO$$/freertos/latest-v10.X
mtb://retarget-io#latest-v1.X#$$ASSET_REPO$$/retarget-io/latest-v1.X

# WiFi + networking
mtb://wifi-connection-manager#latest-v3.X#$$ASSET_REPO$$/wifi-connection-manager/latest-v3.X
mtb://lwip#latest-v2.X#$$ASSET_REPO$$/lwip/latest-v2.X
mtb://mbedtls#latest-v3.X#$$ASSET_REPO$$/mbedtls/latest-v3.X
mtb://secure-sockets#latest-v3.X#$$ASSET_REPO$$/secure-sockets/latest-v3.X

# MQTT
mtb://mqtt#latest-v4.X#$$ASSET_REPO$$/mqtt/latest-v4.X

# HTTP
mtb://http-client#latest-v1.X#$$ASSET_REPO$$/http-client/latest-v1.X

# BLE
mtb://btstack-integration#latest-v5.X#$$ASSET_REPO$$/btstack-integration/latest-v5.X
mtb://bluetooth-freertos#latest-v5.X#$$ASSET_REPO$$/bluetooth-freertos/latest-v5.X

# Sensors
mtb://sensor-motion-bmi270#latest-v1.X#$$ASSET_REPO$$/sensor-motion-bmi270/latest-v1.X

# Emulated EEPROM (data persistence)
mtb://emeeprom#latest-v2.X#$$ASSET_REPO$$/emeeprom/latest-v2.X
```

---

## Required COMPONENTS by Library

| Library | COMPONENTS | Where to add |
|---------|-----------|--------------|
| FreeRTOS | `FREERTOS` | App Makefile |
| lwIP | `LWIP` | App Makefile |
| Mbed TLS | `MBEDTLS` | App Makefile |
| BTSTACK | `BLUETOOTH` | App Makefile |

Add to the correct Makefile:
- **Single-project:** root `Makefile`
- **PSOC Edge:** `proj_cm33_ns/Makefile`

---

## Required DEFINES by Library

| Library/Feature | DEFINES | Notes |
|----------------|---------|-------|
| WiFi-capable kit | `CYBSP_WIFI_CAPABLE` | Required for WiFi init |
| retarget-io newline | `CY_RETARGET_IO_CONVERT_LF_TO_CRLF` | Converts \n → \r\n for terminals |
| Custom heap size | `configTOTAL_HEAP_SIZE=value` | FreeRTOS heap (default varies by BSP) |

---

## Device Family Compatibility

| Library | PSOC Edge | PSOC 6 | PSOC C3 | PSOC 4 | XMC4000 | XMC7000 |
|---------|-----------|--------|---------|--------|---------|---------|
| FreeRTOS | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| retarget-io | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ |
| WiFi CM | ✓ | ✓ | — | — | — | — |
| MQTT | ✓ | ✓ | — | — | — | — |
| lwIP | ✓ | ✓ | — | — | — | — |
| Mbed TLS | ✓ | ✓ | ✓ | — | — | ✓ |
| BTSTACK | ✓ | ✓ | — | — | — | — |
| HTTP Client | ✓ | ✓ | — | — | — | — |
| Emulated EEPROM | ✓ | ✓ | ✓ | ✓ | — | ✓ |

*PSOC Edge retarget-io requires `--wrap=_write` linker flag. See `#retarget-io-fix`.

---

## Known Gotchas

1. **WiFi libraries require CYW43xxx/CYW55xxx radio** — not all kits have WiFi hardware
2. **BTSTACK v3 vs v4 API** — newer BSPs use v4; check `btstack-integration` README for migration
3. **Mbed TLS config** — default config may not enable all cipher suites. Custom `mbedtls_user_config.h` may be needed
4. **FreeRTOS heap** — WiFi + MQTT + TLS can require 80-100 KB heap. Monitor with `xPortGetFreeHeapSize()`
5. **Library version conflicts** — if two libraries pin different versions of a shared dependency, `make getlibs` will warn. Use consistent version tags.
