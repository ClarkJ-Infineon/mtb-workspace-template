---
description: "Add a new ModusToolbox library dependency — fetches COMPONENTS/DEFINES from the library's GitHub README"
---

Add a new ModusToolbox™ library to this project. Follow these steps exactly:

1. **Identify the library repo** on GitHub:
   - URL pattern: `https://github.com/Infineon/[library-name]`
   - If the library name is uncertain, search: `https://github.com/Infineon/?q=[keyword]`

2. **Read the library's README** at `https://github.com/Infineon/[library-name]/blob/master/README.md`:
   - Find the required **COMPONENTS** entry (e.g., `COMPONENTS+=LWIP`)
   - Find any required or useful **DEFINES** (e.g., `DEFINES+=CY_RETARGET_IO_CONVERT_LF_TO_CRLF`)
   - Note any prerequisite libraries that must also be added

3. **Check the library's `docs/` folder** for additional configuration options:
   - `https://github.com/Infineon/[library-name]/tree/master/docs`

4. **Determine the correct `deps/` location** based on the target device (from `CONTEXT.md`):
   - PSOC Edge: `proj_cm33_ns/deps/`
   - All other families: `deps/` at project root

5. **Create the `.mtb` dependency file**:
   ```
   mtb://[library-name]#latest-vX.X#$$ASSET_REPO$$/[library-name]/latest-vX.X
   ```
   File name: `[library-name].mtb` in the correct `deps/` directory.

6. **Update the app Makefile** with COMPONENTS and DEFINES found in step 2:
   - PSOC Edge: `proj_cm33_ns/Makefile`
   - All others: `Makefile` at project root

7. **Check for prerequisite libraries** — if the README lists dependencies, add those `.mtb` entries too.

8. **Summarize what was added**:
   - `.mtb` file path and content
   - Makefile changes (COMPONENTS and DEFINES)
   - Any prerequisite libraries added
   - Link to the library README for the developer's reference

9. **Remind the developer** to run `make getlibs` to download the library before building.

Do not invent COMPONENTS or DEFINES names — only use values found in the library's README or docs.
