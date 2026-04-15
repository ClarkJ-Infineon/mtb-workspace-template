---
name: readme
description: Generate a publication-ready README.md following the standard Infineon code example format. Use when creating or updating the project README, or when the user asks for project documentation.
---

# Generate Project README

Generate a complete, publication-ready README.md for this ModusToolbox™ project.

> **Prerequisites:** Read `CONTEXT.md` and ensure your project has source code to analyze.

---

## Instructions for Copilot

Analyze the project's source code, Makefile, and CONTEXT.md, then generate a README following the standard Infineon code example format below.

### Required Sections

```markdown
# [Project Title]

[One paragraph: what the project does, which hardware it targets, key features]

## Requirements

- [ModusToolbox™ version from CONTEXT.md]
- Programming language: C
- Associated parts: [device from CONTEXT.md]

## Supported toolchains (make variable 'TOOLCHAIN')

- GNU Arm® Embedded Compiler v[version] (`GCC_ARM`) — Default

## Supported kits (make variable 'TARGET')

- [Kit name and ID from CONTEXT.md]

## Hardware setup

[Describe what physical connections are needed. Include any external sensors, shields, or jumper settings. If just the dev kit with USB, say so.]

## Software setup

[Any host-side tools needed: serial terminal settings (115200 8N1), MQTT broker, mobile app, etc.]

## Using the code example

### In Eclipse IDE for ModusToolbox™

1. Click **New Application** in the Quick Panel
2. Select the target kit
3. Select this code example
4. Click **Finish**

### In command-line interface (CLI)

```bash
make getlibs
make build
make program
```

## Operation

[Step-by-step: what happens when the firmware runs. Include expected serial terminal output as a code block.]

## Design and implementation

[Technical overview: architecture, task structure, key algorithms, communication flow. Include a block diagram if helpful.]

### Project structure
[List key source files and their purpose]

### Resources and settings
[Key Makefile COMPONENTS, DEFINES, important config values]

## Related resources

- [ModusToolbox™ Software](https://www.infineon.com/modustoolbox)
- [PSOC Edge documentation](https://www.infineon.com/psocedge) (adjust for target family)

---
© [year] Infineon Technologies AG. All rights reserved.
```

### Guidelines

- **Tone:** Technical, direct, no marketing fluff
- **PSOC** — ALL CAPS always
- **ModusToolbox™** — trademark on first use
- **Serial output examples:** Include actual or representative terminal output
- **Accuracy:** Only describe features the code actually implements — do not embellish
- **Block diagram:** Describe in text what a diagram would show (ASCII art acceptable)
- **Length:** Aim for completeness without padding — typically 150-300 lines
