---
name: openocd-debug
description: Debug ModusToolbox applications using OpenOCD and GDB. Covers make debug/attach workflow, multi-core debugging on PSOC Edge E84 (CM33 + CM55), VS Code launch configuration, CFSR fault register analysis, and common debugging techniques for HardFault, BusFault, and MemManage faults.
---

# OpenOCD / GDB Debugging for ModusToolbox

## Quick Start — Two-Terminal Debug

The ModusToolbox build system provides built-in debug targets:

```bash
# Terminal 1: Build, program, and start OpenOCD GDB server
make debug

# Terminal 2: Launch GDB client and connect to OpenOCD
make attach
```

**`make debug`** = `make program` + starts OpenOCD as GDB server on port 3333
**`make qdebug`** = same but skips rebuild (quick debug)
**`make attach`** = launches `arm-none-eabi-gdb` with the correct `.elf` and `gdbinit` script

> **Build config:** Use `CONFIG=Debug` (the default) for debug symbols. `CONFIG=Release` strips debug info and optimizes.

---

## 1. Tool Locations

OpenOCD and GDB paths are auto-resolved by the build system:

| Tool | Typical location | Build variable |
|---|---|---|
| OpenOCD | `ModusToolboxProgtools-<ver>/openocd/bin/openocd.exe` | `CY_TOOL_openocd_EXE_ABS` |
| OpenOCD scripts | `ModusToolboxProgtools-<ver>/openocd/scripts/` | `CY_TOOL_openocd_scripts_SCRIPT_ABS` |
| GDB | `mtb-gcc-arm-eabi/<ver>/gcc/bin/arm-none-eabi-gdb.exe` | Auto from toolchain |
| Target configs | `openocd/scripts/target/infineon/pse84.cfg` | Auto from BSP |
| Interface | `openocd/scripts/interface/kitprog3.cfg` | Auto from BSP |

Check `build/mtbquery-info.mk` for exact resolved paths in your project.

---

## 2. VS Code Debug Setup

Generate VS Code debug configurations:

```bash
make vscode
```

This creates `.vscode/launch.json` with pre-configured debug targets for each core. The recipe-make generates OpenOCD TCL scripts that configure both CM33 and CM55 with FreeRTOS-aware debugging.

---

## 3. PSOC Edge Multi-Core Debugging

### OpenOCD multi-core configuration (auto-generated)

The recipe-make's TCL script configures:

```tcl
set ENABLE_CM55 1
source [find target/pse84.cfg]
pse84.cm33 configure -rtos auto -rtos-wipe-on-reset-halt 1
pse84.cm55 configure -rtos auto -rtos-wipe-on-reset-halt 1
gdb_breakpoint_override hard
```

### GDB port mapping

| Core | GDB port | How to connect |
|---|---|---|
| CM33 (default) | 3333 | `target remote :3333` |
| CM55 | 3334 | `target remote :3334` |

### Debugging a specific core

**CM33 NS** (default — `make attach` connects here):
```
(gdb) target remote :3333
(gdb) monitor targets pse84.cm33
```

**CM55:**
```
(gdb) target remote :3334
(gdb) monitor targets pse84.cm55
```

### Loading symbols for the correct core

```
# For CM33 NS debugging:
(gdb) file proj_cm33_ns/build/APP_KIT_PSE84_EVAL_EPC2/Debug/proj_cm33_ns.elf

# For CM55 debugging:
(gdb) file proj_cm55/build/APP_KIT_PSE84_EVAL_EPC2/Debug/proj_cm55.elf
```

---

## 4. GDB gdbinit Script (auto-loaded by `make attach`)

The recipe-make provides a `gdbinit` script that runs automatically:

```gdb
target remote :3333
set mem inaccessible-by-default off
monitor arm semihosting enable
monitor reset init
flushregs
mon gdb_sync
stepi
tbreak main
monitor reg
continue
```

This connects to CM33, enables semihosting, resets to `main()`, and continues.

---

## 5. Common Debug Commands

### Breakpoints and execution

```gdb
break main.c:42          # Line breakpoint
break wifi_task           # Function breakpoint
watch my_variable         # Data watchpoint (stops when value changes)
continue                  # Resume execution
step / next              # Step into / step over
finish                   # Run until current function returns
```

### Memory and registers

```gdb
monitor reg              # Print all CPU registers
print /x $pc            # Program counter
print /x $sp            # Stack pointer
print /x $lr            # Link register (return address)

x/16xw 0x240FEC00       # Examine 16 words at IPC address
x/s 0x20000000          # Examine string at address
```

### FreeRTOS-aware commands (with `-rtos auto`)

```gdb
info threads            # List all FreeRTOS tasks
thread 2                # Switch to task 2
bt                      # Backtrace for current task
```

---

## 6. HardFault / BusFault Diagnosis

### CFSR — Configurable Fault Status Register (0xE000ED28)

```gdb
# Read fault registers
print /x *(uint32_t*)0xE000ED28    # CFSR (combined)
print /x *(uint32_t*)0xE000ED34    # HFSR (HardFault Status)
print /x *(uint32_t*)0xE000ED38    # DFSR (Debug Fault Status)
print /x *(uint32_t*)0xE000ED3C    # MMFAR (MemManage Fault Address)
print /x *(uint32_t*)0xE000ED38    # BFAR (BusFault Address)
```

### CFSR bit decode

| Bit | Name | Meaning |
|---|---|---|
| [0] IACCVIOL | Instruction access violation | Code fetch from non-executable region |
| [1] DACCVIOL | Data access violation | Read/write to protected region (PPU!) |
| [7] MMARVALID | MMFAR valid | MMFAR contains the fault address |
| [8] IBUSERR | Instruction bus error | Flash/code fetch failure |
| [9] PRECISERR | Precise data bus error | BFAR has exact fault address |
| [10] IMPRECISERR | Imprecise data bus error | Fault address not available |
| [15] BFARVALID | BFAR valid | BFAR contains the fault address |
| [16] UNDEFINSTR | Undefined instruction | Usually alignment or corruption |
| [17] INVSTATE | Invalid state | Thumb/ARM mismatch |
| [18] INVPC | Invalid PC | Bad exception return |
| [24] UNALIGNED | Unaligned access | Common with DMA buffers |

### Common fault patterns on PSOC Edge

| CFSR value | Likely cause |
|---|---|
| `0x00008200` (PRECISERR + BFARVALID) | PPU blocking access — check BFAR address |
| `0x00000001` (IACCVIOL) | Code running from non-executable region |
| `0x00000082` (DACCVIOL + MMARVALID) | MPU violation — shared memory not configured |
| `0x00010000` (UNDEFINSTR) | Stack corruption or bad function pointer |

### PPU (Protection Policy Unit) on PSOC Edge

CM55 cannot access peripherals reserved for CM33 (e.g., SCB2/UART). A PPU violation looks like a BusFault with BFAR pointing to the peripheral address. **There is no runtime error message** — only a fault. Check the BFAR address against the memory map.

---

## 7. Debugging D-Cache Issues

Shared memory data appears stale or corrupted between cores:

```gdb
# Examine raw memory at shared address
x/8xw 0x240FEC00

# Check if D-cache is enabled
print /x *(uint32_t*)0xE000EF50    # CCR register, bit 16 = DC
```

**Fix:** Use `CY_SECTION_SHAREDMEM` for shared data (maps to non-cacheable region), or add explicit cache maintenance:

```c
SCB_InvalidateDCache_by_Addr((void*)shared_ptr, sizeof(*shared_ptr));
```

---

## 8. OpenOCD Direct Commands

When `make debug` is running, you can issue OpenOCD commands via GDB's `monitor` prefix:

```gdb
monitor reset halt          # Reset and halt
monitor reset init          # Reset and run init scripts
monitor reset run           # Reset and resume
monitor halt                # Halt CPU
monitor resume              # Resume CPU
monitor reg                 # Dump all registers
monitor targets             # List available debug targets
monitor flash info 0        # Flash bank information
monitor adapter speed 1000  # Set probe speed (KHz)
```

---

## 9. Common Pitfalls

1. **`make debug` hangs at "Opening GDB port"** — Normal. Open a second terminal and run `make attach`. OpenOCD waits for a GDB connection.

2. **"Error: Can't find a KitProg3 device"** — Board not connected, or another tool (MTB Programmer, IDE) holds the probe. Close other debug tools first.

3. **Debugging CM55 but symbols wrong** — `make attach` defaults to CM33. For CM55, connect to port 3334 and load the CM55 `.elf` file manually.

4. **Breakpoints not hitting** — May need hardware breakpoints. OpenOCD auto-sets `gdb_breakpoint_override hard`. Verify with `info breakpoints`.

5. **FreeRTOS tasks not showing** — RTOS-aware debugging requires `-rtos auto` in OpenOCD config (auto-configured by recipe-make) and FreeRTOS symbols in the `.elf`.

6. **"Target not halted" errors** — Issue `monitor halt` before memory reads or register inspection.

7. **Post-mortem debugging** — If the device faulted, `monitor halt` + read CFSR/BFAR/MMFAR before reset. The fault registers are cleared on reset.
