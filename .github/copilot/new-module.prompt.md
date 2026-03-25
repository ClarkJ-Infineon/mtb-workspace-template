---
description: "Scaffold a new .c/.h module pair with correct Doxygen file headers for the target device"
---

Create a new source module (`.c` and `.h` file pair) for this project.

1. **Read `CONTEXT.md`** to get:
   - Target device family and specific device
   - Kit name
   - ModusToolbox™ version

2. **Determine the correct file locations** based on target device (from `CONTEXT.md`):
   - PSOC Edge: `proj_cm33_ns/source/[module-name].c` and `proj_cm33_ns/include/[module-name].h`
   - All other families: `source/[module-name].c` and `include/[module-name].h`

3. **Create the `.h` file** with:
   - File header (see template below)
   - Include guard (`#ifndef`, `#define`, `#endif`)
   - Required includes (`stdint.h`, `stdbool.h`, device BSP header)
   - `#ifdef __cplusplus` / `extern "C"` block for C++ compatibility
   - Doxygen `@defgroup` block for the module
   - Public type definitions and struct declarations
   - Public function prototypes with full Doxygen (`@brief`, `@param`, `@return`)
   - Any public `#define` macros with comments explaining value, units, and safe range

4. **Create the `.c` file** with:
   - File header (see template below)
   - Include for its own `.h` file first, then other dependencies
   - Private (file-scope) defines and variables with `/** @brief */` comments
   - Implementation of all functions declared in the header
   - Error handling that matches the project's convention (check `CONTEXT.md` Non-Negotiable Rules)

**File header template** (use device and kit values from `CONTEXT.md`):
```c
/******************************************************************************
 * File Name: [filename]
 * Description: [One sentence — what this module does]
 *
 * Target: [device family + specific device]
 * Kit: [kit name]
 * ModusToolbox™ version: [from CONTEXT.md]
 *
 * SPDX-License-Identifier: Apache-2.0
 * Copyright (c) [current year] Infineon Technologies AG. All rights reserved.
 ******************************************************************************/
```

**Include guard template** (`.h` files only):
```c
#ifndef [FILENAME_H]
#define [FILENAME_H]

// ... content ...

#endif /* [FILENAME_H] */
```

5. **After creating the files**, remind the developer to:
   - Add `source/[module-name].c` to the build (it will be picked up automatically if in the `source/` directory and the Makefile uses the default `SOURCES` glob)
   - Add `include/[module-name].h` to the include path if not already covered by the Makefile `INCLUDES` setting
