# Sesame Robot - project notes

The standard work process from
[Dorus-Claude-Collaboration](https://github.com/DorusvdLinden/Dorus-Claude-Collaboration)
(synced into `~/.claude/CLAUDE.md`) applies here by default: working
modes (Plan/Build/Away), branch-before-editing, verification discipline,
the four-doc set (README/TODO/IDEAS/CHANGES), artifact preservation,
etc. This file only adds what's specific to this project.

## Repo relationship

- `origin` = `Dorus-Robots/sesame-robot` (this fork).
- `upstream` = `dorianborian/sesame-robot` (original project). Pull from
  `upstream` when checking whether an upstream fix/feature applies here,
  rather than assuming this fork is current with it.
- The `main` branch is adapted to the **Sesame Distro Board V2/V3
  (ESP32-S3)**: I2C on GPIO 8/9, `servoPins[8]` set to the Distro V2
  pinout. See `firmware/sesame-firmware-main/sesame-firmware-main.ino`
  and `firmware/README.md`'s pin-configuration tables before assuming a
  different hardware revision's pinout applies.

## Hardware

- 4 legs, 2 servos/leg (8 total), direct-GPIO PWM via `ESP32Servo` (no
  PCA9685 or other I2C PWM expander in this codebase).
- No IK — every pose/gait in `movement-sequences.h` is hand-authored
  with literal per-servo angles. See that file's `ServoName` enum
  (`R1/R2/L1/L2/R3/R4/L3/L4`) for the servo-naming scheme.

## Related project

A 5-legged variant ("Rocky") was designed starting from this codebase's
research but lives in its own repo:
[rocky-robot](https://github.com/DorusvdLinden/rocky-robot). This repo's
copy of that planning work is at `plans/rocky-robot-design.md` and
`plans/rocky-robot-todo.md` (see `CHANGES.md` entry 0002) - kept here as
a record of what was researched from this codebase, but the live/current
version of that plan is in the rocky-robot repo itself.
