# Changes

Numbered log of key decisions, with current status.

## 0001 - Forked from dorianborian/sesame-robot, adapted to own hardware

Status: active

This repo (`Dorus-Robots/sesame-robot`) is a fork of the upstream
project (tracked as the `upstream` remote), adapted to run on the Sesame
Distro Board V2/V3 (ESP32-S3): I2C pins repointed to SDA=8/SCL=9 and the
`servoPins[8]` array switched to the Distro V2 pinout
(`2a7a9b5 "adapted to own hardware"`). No structural changes to the
4-leg/2-servo-per-leg design have been made.

## 0002 - Rocky robot spun off as a separate sibling repo

Status: active

A 5-legged, 15-servo variant ("Rocky", inspired by *Project Hail Mary*)
was designed starting from this codebase but built out as its own repo,
[rocky-robot](https://github.com/DorusvdLinden/rocky-robot), rather than
as a variant inside this one — the leg count, servo count, and drive
electronics (PCA9685 vs this repo's direct-GPIO ESP32Servo) diverge
enough that keeping both in one repo would add conditional complexity
without much shared code beyond the WiFi/web/face scaffolding, which was
copied over as a starting template instead of shared at the code level.
This repo's own planning research for that spinoff is kept at
`plans/rocky-robot-design.md` and `plans/rocky-robot-todo.md`.
