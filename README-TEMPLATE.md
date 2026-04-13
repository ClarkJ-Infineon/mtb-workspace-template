# README Template — ModusToolbox Code Example

> Copy this file to your project root and rename it to `README.md`. Fill in each section.
> Delete this instruction block and any sections that don't apply to your project.

# [Project Title — e.g., "PSOC Edge: MQTT Sensor Hub with BMI270"]

[One paragraph describing what this project does, which hardware it targets, and the key feature or learning objective.]

## Requirements

- [ModusToolbox™ version — from CONTEXT.md]
- Programming language: C
- Associated parts: [device — from CONTEXT.md]

## Supported toolchains (make variable 'TOOLCHAIN')

- GNU Arm® Embedded Compiler v11.3.1 (`GCC_ARM`) — Default

## Supported kits (make variable 'TARGET')

- [Kit name and ID — from CONTEXT.md, e.g., "PSOC Edge E84 Evaluation Kit (`KIT_PSE84_EVAL_EPC2`)"]

## Hardware setup

[Describe what's needed beyond the bare dev kit: external sensors, shields, jumper settings, antenna orientation. If nothing extra, state "This code example requires only the [kit name] kit."]

## Software setup

[Host-side tools: serial terminal (115200 8N1), MQTT broker URL, mobile app, Python script, etc.]

Install a serial terminal emulator (e.g., PuTTY, Tera Term, or minicom) and connect to the KitProg3 COM port at **115200 baud, 8N1**.

## Using the code example

### In Eclipse IDE for ModusToolbox™

1. Click the **New Application** link in the **Quick Panel** (or, use **File** > **New** > **ModusToolbox™ Application**)
2. Select your kit from the list
3. Select this application from the list
4. Click **Finish**

### In command-line interface (CLI)

```bash
make getlibs
make build
make program
```

## Operation

[Step-by-step what happens after programming. Include representative serial terminal output:]

```
[CM33] cybsp_init complete
[CM33] WiFi connected: 192.168.1.42
[CM33] MQTT connected to broker
[CM33] Publishing sensor data...
```

## Design and implementation

[Technical overview of the firmware architecture. Cover:]
- [Task structure (FreeRTOS tasks and their responsibilities)]
- [Communication flow (sensor → processing → output)]
- [Key algorithms or protocols used]

### Project structure

| File | Description |
|------|-------------|
| `main.c` | Application entry point, task creation |
| `source/sensor_task.c` | Sensor data acquisition |
| `include/app_config.h` | Configuration defines (WiFi, MQTT, thresholds) |

### Resources and settings

| Resource | Details |
|----------|---------|
| COMPONENTS | `FREERTOS`, `LWIP`, `MBEDTLS` |
| DEFINES | `CYBSP_WIFI_CAPABLE`, `CY_RETARGET_IO_CONVERT_LF_TO_CRLF` |
| Debug UART | 115200 baud, KitProg3 |

## Related resources

- [ModusToolbox™ software environment](https://www.infineon.com/modustoolbox)
- [Infineon PSOC Edge developer resources](https://www.infineon.com/psocedge)

---

© 2026 Infineon Technologies AG. All rights reserved.
