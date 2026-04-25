---
name: build-error
description: Diagnose ModusToolbox build failures by checking common MTB-specific root causes. Use this when a build fails, compiler or linker errors appear, undefined symbol errors occur, or when the user shares a build error message.
---

Diagnose the ModusToolbox™ build error provided. Work through the checklist below in order — most MTB build failures fall into one of these categories.

Read `CONTEXT.md` first to confirm the target device family, kit, and toolchain before diagnosing.

---

## MTB Build Error Diagnosis Checklist

### 1. Missing `make getlibs`
**Symptoms:** `No such file or directory` for a library header; `undefined reference` to a library function.
**Check:** Does the `libs/` directory exist and contain the expected library folders?
**Fix:** Run `make getlibs` in the project root (or the app sub-project for PSOC Edge).

### 2. Missing COMPONENTS entry
**Symptoms:** `undefined reference` to a middleware function (e.g., FreeRTOS, lwIP, Mbed TLS).
**Check:** Does the app Makefile contain the required `COMPONENTS+=` line?
**Reference:** Check the library's GitHub README at `https://github.com/Infineon/[library-name]` for the required COMPONENTS entry.
**Fix:** Add the missing `COMPONENTS+=` line to the Makefile.

### 3. Wrong HAL API for the target device
**Symptoms:** `undefined reference` to `cyhal_*` functions on PSOC Edge or PSOC Control C3; or `undefined reference` to `mtb_hal_*` on PSOC 6.
**Check:** Compare the API prefix in the failing code against the device family in `CONTEXT.md`:
- PSOC Edge / PSOC Control C3 → use `mtb_hal_` (from DSL or `mtb-hal-psc3`)
- PSOC 6 / XMC7000 → use `cyhal_` (from `mtb-hal-cat1`)
- PSOC 4 → use `cyhal_` (from `mtb-hal-cat2`, limited coverage)
- XMC1000 / XMC4000 → use `XMC_` (from `mtb-xmclib-cat3`; no HAL)
**Fix:** Replace API calls with the correct HAL prefix for the target device.

### 4. Wrong PDL headers (CAT1 vs CAT2)
**Symptoms:** Compilation errors in PDL driver code; type mismatches.
**Check:** Is the code using CAT1 PDL (`mtb-pdl-cat1`) on a PSOC 4 target that needs CAT2 PDL (`mtb-pdl-cat2`)?
**Fix:** Use the correct PDL package for the device family.

### 5. Wrong TARGET in Makefile
**Symptoms:** BSP-related errors; missing `cybsp.h` or `cycfg_*.h`; `make program` targeting wrong device.
**Check:** Does the `TARGET=` value in `common.mk` (or `Makefile`) match the kit in `CONTEXT.md`?
**Fix:** Update the `TARGET=` line to the correct kit name (e.g., `KIT_PSE84_EVAL_EPC2`).

### 6. PSOC Edge — code in wrong sub-project
**Symptoms:** Build succeeds but application does not run; or linker errors about missing symbols across sub-projects.
**Check:** Is application logic in `proj_cm33_s/` or `proj_cm55/` instead of `proj_cm33_ns/`?
**Fix:** Move all application code to `proj_cm33_ns/source/` and `proj_cm33_ns/include/`.

### 7. GeneratedSource/ out of sync
**Symptoms:** Errors in `GeneratedSource/` files; missing peripheral init functions.
**Check:** Has the Device Configurator been saved and code regenerated after a hardware change?
**Fix:** Open `design.modus` in the MTB IDE, make no changes, save — this regenerates `GeneratedSource/`.

### 8. Toolchain-specific issue
**Symptoms:** Errors only with one TOOLCHAIN (GCC_ARM, ARM, LLVM).
**Check:** Does the code use compiler-specific intrinsics or attributes without an `#ifdef` guard?
**Fix:** Add appropriate guards (e.g., `#if defined(__ARM_FEATURE_MVE)` for Helium intrinsics).

### 9. Project not created via project-creator-cli
**Symptoms:** Multiple undefined symbols for BSP types (`GFXSS_Type`, `CYBSP_*`), missing `$(SEARCH_*)` variables, linker script errors.
**Check:** Was the project created via `project-creator-cli` or was it manually assembled?
**Fix:** Use the /project-creation skill to create the project correctly. Manual project assembly is not a supported workflow.

### 10. Memory region overflow (PSOC Edge linker error)
**Symptoms:** Linker error mentioning `.data` or `.bss` section overflow; region `m33_data` or `m55_data` overflowed.
**Check:** Run `arm-none-eabi-size` on the .elf to see actual usage vs. region capacity.
**Fix:** Open `design.modus` → Memory Configuration → rebalance regions. Common fix: shrink `m33_code` (typically underutilized), grow `m33_data`. See /dual-core-setup skill for memory rebalancing guidance.

### 11. Clock configuration mismatch
**Symptoms:** UART baud rate wrong (garbled output), timers run at wrong speed, performance doesn't match expectations.
**Check:** Verify actual clock frequencies from `bsps/TARGET_*/config/GeneratedSource/cycfg_clocks.c` — search for `desiredFrequency` and `CLKHF` defines. CM55 may be at 400 MHz, not 240 MHz as sometimes documented.
**Fix:** Adjust peripheral clock dividers in Device Configurator → Clocks tab, or update application code to use the correct frequency constants.

### 12. Device Configurator changes silently lost?
**Symptoms:** You save changes in Device Configurator but the generated code doesn't reflect them; peripheral behavior unchanged after rebuild.
**Check:** Look for a stale `design.modus.lock` file in the BSP config directory.
**Fix:** Delete `design.modus.lock`, reopen Device Configurator, and retry. Verify the `.modus` file modification timestamp actually changes after saving.

### 13. Memory Configurator rejects region resize with overlap error?
**Symptoms:** Memory Configurator shows overlap error when resizing a memory region, even though the final layout is valid.
**Fix:** Resize in two save cycles: (1) shrink the donor region and save, (2) grow the target region and save. The tool validates intermediate states — you cannot shrink and grow in a single operation if the intermediate state would cause overlap.

### 14. IPC data corruption or D-Cache coherency bugs after memory rebalancing?
**Symptoms:** IPC returns stale/zero data; shared memory corruption appears only after Memory Configurator changes.
**Check:** Compare MPU regions in the generated `cycfg_system.c` against the new memory layout in `cymem_CM55_0.h` (or `cymem_CM33_0.h`).
**Fix:** Manually verify that every MPU region start address and size matches the Memory Configurator output. Device Configurator does NOT auto-sync MPU addresses with Memory Configurator — this is a known limitation.

### 15. Build fails with warnings treated as errors?
**Symptoms:** Compiler warnings (`-Wunused-variable`, `-Wimplicit-int-conversion`, etc.) cause build failure.
**Check:** Matter SDK projects and some Infineon templates enable `-Werror`. Check project and library Makefiles for `CFLAGS += -Werror`.
**Fix:** Fix ALL compiler warnings — unused variables, implicit type conversions, missing return statements all become fatal under `-Werror`. Do not remove the flag; fix the code.

---

After identifying the root cause, provide:
1. **Root cause:** [one sentence]
2. **Fix:** exact change needed (file, line, what to add/change)
3. **Verification:** how to confirm the fix worked (`make build` output to look for)
