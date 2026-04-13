---
description: "Add a new ModusToolbox library dependency — fetches COMPONENTS/DEFINES from the library's GitHub README"
---

Add a new ModusToolbox™ library to this project. Follow these steps exactly:

1. **Identify the library repo** on GitHub:
   - URL pattern: `https://github.com/Infineon/[library-name]`
   - If the library name is uncertain, search: `https://github.com/Infineon/?q=[keyword]`
   - Master library index: `https://github.com/Infineon/modustoolbox-software/blob/master/README.md`

2. **Read the library's README** at `https://github.com/Infineon/[library-name]/blob/master/README.md`:
   - Find the required **COMPONENTS** entry (e.g., `COMPONENTS+=LWIP`)
   - Find any required or useful **DEFINES** (e.g., `DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF`)
   - Note any prerequisite libraries that must also be added

3. **Check the library's `docs/` folder** for additional configuration options:
   - `https://github.com/Infineon/[library-name]/tree/master/docs`

4. **Determine the correct `deps/` location** based on the target device (from `CONTEXT.md`):
   - PSOC Edge: `proj_cm33_ns/deps/` (or `proj_cm55/deps/` if library is for CM55)
   - All other families: `deps/` at project root

5. **Create the `.mtb` dependency file**:
   ```
   mtb://[library-name]#latest-vX.X#$$ASSET_REPO$$/[library-name]/latest-vX.X
   ```
   File name: `[library-name].mtb` in the correct `deps/` directory.

   **Version tag conventions:**
   - `latest-vX.X` — latest patch within major.minor (recommended)
   - `release-vX.X.X` — pinned to exact version (reproducible builds)

6. **Update the app Makefile** with COMPONENTS and DEFINES found in step 2:
   - PSOC Edge: `proj_cm33_ns/Makefile`
   - All others: `Makefile` at project root

7. **Check for prerequisite libraries** — if the README lists dependencies, add those `.mtb` entries too. Common dependency chains:
   - WiFi + MQTT requires: `wifi-connection-manager`, `mqtt`, `secure-sockets`, `lwip`, `mbedtls`, `freertos`
   - BLE requires: `btstack-integration`, `bluetooth-freertos`, `freertos`
   - HTTP requires: `http-client`, `secure-sockets`, `lwip`, `mbedtls`, `freertos`

8. **Summarize what was added**:
   - `.mtb` file path and content
   - Makefile changes (COMPONENTS and DEFINES)
   - Any prerequisite libraries added
   - Link to the library README for the developer's reference

9. **Remind the developer** to run `make getlibs` to download the library before building.

Do not invent COMPONENTS or DEFINES names — only use values found in the library's README or docs.

---

## Common Libraries Quick Reference

> **Always verify** against the library's current GitHub README — versions and config may change.

| Library | .mtb entry | COMPONENTS | Families |
|---------|-----------|------------|---------|
| FreeRTOS | `mtb://freertos#latest-v10.X#$$ASSET_REPO$$/freertos/latest-v10.X` | `FREERTOS` | All |
| retarget-io | `mtb://retarget-io#latest-v1.X#$$ASSET_REPO$$/retarget-io/latest-v1.X` | — | All |
| WiFi CM | `mtb://wifi-connection-manager#latest-v3.X#$$ASSET_REPO$$/wifi-connection-manager/latest-v3.X` | — | PSOC 6, Edge |
| lwIP | `mtb://lwip#latest-v2.X#$$ASSET_REPO$$/lwip/latest-v2.X` | `LWIP` | PSOC 6, Edge |
| Mbed TLS | `mtb://mbedtls#latest-v3.X#$$ASSET_REPO$$/mbedtls/latest-v3.X` | `MBEDTLS` | PSOC 6, Edge |
| Secure Sockets | `mtb://secure-sockets#latest-v3.X#$$ASSET_REPO$$/secure-sockets/latest-v3.X` | — | PSOC 6, Edge |
| MQTT | `mtb://mqtt#latest-v4.X#$$ASSET_REPO$$/mqtt/latest-v4.X` | — | PSOC 6, Edge |
| HTTP Client | `mtb://http-client#latest-v1.X#$$ASSET_REPO$$/http-client/latest-v1.X` | — | PSOC 6, Edge |
| BTSTACK | `mtb://btstack-integration#latest-v5.X#$$ASSET_REPO$$/btstack-integration/latest-v5.X` | `BLUETOOTH` | PSOC 6, Edge |
| BMI270 Sensor | `mtb://sensor-motion-bmi270#latest-v1.X#$$ASSET_REPO$$/sensor-motion-bmi270/latest-v1.X` | — | All |
| Emulated EEPROM | `mtb://emeeprom#latest-v2.X#$$ASSET_REPO$$/emeeprom/latest-v2.X` | — | PSOC 6, Edge, C3 |

See `reference/middleware-compatibility.md` for full dependency chains and device compatibility.
