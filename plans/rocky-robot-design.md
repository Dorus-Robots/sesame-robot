# Rocky Robot — Design Plan

Status: draft, awaiting confirmation before implementation begins (Mode 1).

## 1. Vision

A 5-legged creature inspired by Rocky from *Project Hail Mary*, built as a
sibling project to [sesame-robot](https://github.com/dorianborian/sesame-robot),
reusing its proven firmware scaffolding (WiFi/web UI, OLED face system,
captive portal, JSON API, serial CLI) while replacing everything tied to
sesame's 4-leg/2-servo-per-leg geometry.

CAD reference (Onshape, not machine-readable from this session — see
[Open Questions](#7-open-questions--blocking-items)):
https://cad.onshape.com/documents/6919ae6fe2d8279e9634202a/w/618004bde3b9cb86abf4748f/e/9e3d53aa40c2a613ed380308

**Topology**: 5 legs arranged with radial (pentaradial) symmetry around a
central body, no natural left/right pairing (unlike sesame's bilateral
quadruped). Each leg has 3 servos, all 180°:

| Joint | Location | Motion | Status |
|---|---|---|---|
| Hip yaw | top of leg | rotates leg in the horizontal plane | exists (carried over from current build) |
| Hip pitch | top of leg | bends the leg up/down (stacked with yaw) | **new** — being added |
| Knee pitch | mid-leg | bends the leg | exists (carried over from current build) |

This is a standard coxa-yaw / femur-pitch / tibia-pitch 3-DOF leg
(mechanically equivalent to a hexapod leg), which is a significant upgrade
over sesame's 2-DOF leg (yaw + single bend) — with 3 DOF each leg can
actually lift and place its foot independently of its sweep angle, which
sesame's geometry can't do. That capability is the whole reason this plan
recommends inverse kinematics instead of copying sesame's hand-authored
pose style (see §5).

Total: **15 servos** (5 × 3), up from sesame's 8.

## 2. Decisions made (confirmed with user)

- **Repo**: new sibling repo, working name `rocky-robot`, seeded by copying
  sesame-robot's directory layout as a starting template. Keeps
  sesame-robot clean for tracking upstream (`dorianborian/sesame-robot`);
  Rocky's CAD/BOM/frame diverge enough that a shared repo would just add
  conditional complexity.
- **Servo driver**: PCA9685 16-channel I2C PWM expander (15 of 16 channels
  used, 1 spare). sesame's own firmware README already flags the ESP32's
  4 hardware PWM timers as a scaling concern at just 8 direct-GPIO servos
  (`ESP32PWM::allocateTimer(0..3)`); at 15 servos this is avoided entirely
  by moving PWM generation off the ESP32. The PCA9685 shares the existing
  I2C bus (already used for the SSD1306 OLED at `0x3C`; PCA9685 defaults
  to `0x40`, no conflict) rather than requiring a second bus.
- **Motion system**: data-driven poses + simple per-leg inverse kinematics,
  instead of sesame's fully hand-authored `setServoAngle()` literals. See
  §5 for the concrete design.

## 3. Electrical architecture

- MCU: ESP32-S3 (matches the Distro Board V2/V3 already in use on the
  current sesame fork — no reason to change families, though the board
  itself will need a new hardware revision to route to a PCA9685 instead
  of direct GPIO servo headers; that's a hardware-track task, not code).
- PCA9685 on the existing I2C bus (`I2C_SDA`/`I2C_SCL`, same pins as
  currently defined for Distro V2/V3 — GPIO 8/9 per
  `sesame-firmware-main.ino`).
- Servo channels 0–14 = the 15 leg servos; channel 15 spare (e.g. future
  head/sensor servo).
- Power: 15 servos will roughly double peak current draw vs sesame's 8 —
  BOM/power-supply sizing needs revisiting once servo model is chosen
  (see Open Questions).

## 4. Code structure (target repo layout)

Mirrors sesame-robot's proven layout; only the pieces that must change
for 5 legs / 3 joints / PCA9685 are called out.

```
rocky-robot/
├── README.md                          # adapted from sesame's, new pitch/specs
├── LICENSE
├── docs/
│   ├── build-guide/                   # rewritten for new frame + PCA9685 wiring
│   └── wiring-guide/
├── firmware/
│   ├── README.md                      # adapted: PCA9685 setup, NUM_SERVOS=15, leg/joint map
│   ├── rocky-firmware-main/
│   │   ├── rocky-firmware-main.ino    # reused from sesame almost as-is:
│   │   │                              #   WiFi/AP, web server, captive portal,
│   │   │                              #   serial CLI, OLED face driver.
│   │   │                              #   Changes: PCA9685 init instead of
│   │   │                              #   ESP32Servo/servoPins[], NUM_SERVOS=15,
│   │   │                              #   subtrim[15], angle->PWM via PCA9685
│   │   │                              #   pulse range instead of attach().
│   │   ├── leg-kinematics.h           # NEW — per-leg IK/FK, link-length &
│   │   │                              #   hip-offset constants, leg-to-body
│   │   │                              #   rotation transform (§5)
│   │   ├── movement-sequences.h       # REWRITTEN — pose table (foot targets,
│   │   │                              #   not joint angles) + gait phase
│   │   │                              #   generator, replaces sesame's
│   │   │                              #   per-servo-literal pose functions
│   │   ├── face-bitmaps.h             # reused as-is (cosmetic decision open,
│   │   │                              #   see §7 — does Rocky even have a face?)
│   │   └── captive-portal.h           # reused as-is; web UI command set
│   │                                  #   (forward/left/right/etc.) unchanged,
│   │                                  #   just now dispatches into the gait
│   │                                  #   generator instead of literal poses
│   └── debugging-firmware/
│       └── rocky-motor-tester.ino     # adapted from sesame-motor-tester.ino:
│                                      #   drive channels via PCA9685, sweep
│                                      #   0-14. First thing to bring up on
│                                      #   real hardware (§6, M1).
├── hardware/
│   ├── cad/                           # from the Onshape doc above
│   ├── pcb/                           # new "Rocky Distro Board" revision:
│   │                                  #   PCA9685 + ESP32-S3 + power in,
│   │                                  #   OR v1 bring-up with an off-the-
│   │                                  #   shelf PCA9685 breakout + existing
│   │                                  #   board, deferring a custom PCB
│   ├── printing/stl/                  # 15 joint prints instead of 8, new
│   │                                  #   hip bracket (now holds 2 stacked
│   │                                  #   servos: yaw + pitch)
│   └── bom/
├── software/
│   └── rocky-studio/                  # v2 idea, not blocking: adapt
│                                      #   sesame_studio.py from a raw
│                                      #   per-servo-angle tool into an
│                                      #   IK-aware pose composer (set foot
│                                      #   X/Y/Z per leg, tool computes
│                                      #   angles). Not required for v1
│                                      #   bring-up — serial CLI is enough.
└── plans/
    └── rocky-robot-design.md          # this document, carried into the new repo
```

## 5. Servo naming & motion system

### 5.1 Addressing scheme

Sesame's flat `enum ServoName { R1, R2, L1, L2, ... }` doesn't generalize
to 5-fold radial symmetry (no left/right). Replace it with a 2D
`(leg, joint)` index:

```cpp
enum Joint : uint8_t { HIP_YAW = 0, HIP_PITCH = 1, KNEE_PITCH = 2 };
#define NUM_LEGS 5
#define JOINTS_PER_LEG 3
#define NUM_SERVOS (NUM_LEGS * JOINTS_PER_LEG)  // 15

// PCA9685 channel wiring map — filled in once during hardware bring-up
const uint8_t servoChannel[NUM_LEGS][JOINTS_PER_LEG] = { ... };

inline uint8_t channelFor(uint8_t leg, Joint j) { return servoChannel[leg][j]; }
```

Legs numbered `LEG0`–`LEG4`, clockwise viewed from above, `LEG0` aligned
with whatever the CAD treats as "front" (default assumption — confirm
against the Onshape model once dimensions are pulled).

### 5.2 Per-leg inverse kinematics

Each leg is a standard coxa/femur/tibia chain:

- `hipOffset` — radius from body center to the hip yaw axis
- `L1` — hip pitch axis to knee pitch axis (femur length)
- `L2` — knee pitch axis to foot tip (tibia length)

All three are CAD-derived constants — currently unknown, see §7.

Given a desired foot target in leg-local Cartesian coordinates `(x, y, z)`:

```
yaw   = atan2(y, x)
r     = sqrt(x*x + y*y) - hipOffset
d     = sqrt(r*r + z*z)
kneeAngle  = acos((L1*L1 + L2*L2 - d*d) / (2*L1*L2))         // law of cosines
hipPitch   = atan2(z, r) + acos((L1*L1 + d*d - L2*L2) / (2*L1*d))
```

(Standard 2-link planar IK in the `(r, z)` plane after removing yaw —
same math used in most hexapod leg controllers.) This becomes
`leg-kinematics.h`'s `solveLegIK(x, y, z) -> (yaw, hipPitch, kneeAngle)`.

### 5.3 Body-level pose symmetry

Define each named pose (stand, rest, reach, wave, etc.) **once** as a
foot target relative to a leg's own hip — not per named servo like
sesame. Applying it to all 5 legs is then just index bookkeeping (no
per-leg literal angles to hand-tune 5 times):

```cpp
FootTarget standTarget = { .x = 60, .y = 0, .z = -40 }; // example, mm

for (int leg = 0; leg < NUM_LEGS; leg++) {
  applyLegIK(leg, standTarget);   // same target, IK handles per-leg geometry
}
```

Left/right mirroring and per-leg hand-authoring — both real costs in
sesame's `movement-sequences.h` — disappear entirely for symmetric poses.
Asymmetric poses (e.g. "wave" with one leg raised) just override that one
leg's target.

### 5.4 Gait

5 legs is an odd number — sesame's implicit diagonal-pair gait doesn't
map over. Recommend a **wave gait** for v1: lift and advance one leg at a
time in sequence (`LEG0 → LEG1 → LEG2 → LEG3 → LEG4`), always 4 legs
grounded — maximizes stability for a first walking version and is the
easiest gait to reason about while tuning. A faster alternating gait
(e.g. 2+3 leg groups) is a reasonable v2 once the wave gait is validated
on hardware.

Implementation: one shared leg-trajectory function (lift → swing → place)
driven by a global gait-phase counter, offset per leg by
`legIndex * (period / NUM_LEGS)`. This directly reuses sesame's
`pressingCheck()` pattern for making the gait loop cancelable/interruptible
from the web UI.

### 5.5 Calibration surface

Keep sesame's per-servo `subtrim[15]` array (same mechanism, resized).
Add the new CAD-derived constants as the second calibration layer:
`hipOffset`, `L1`, `L2` (shared across legs if the build is symmetric),
plus a per-leg mounting-angle constant (`72° * legIndex` nominally, but
make it a table in case the real build isn't perfectly symmetric).

## 6. Bring-up milestones

1. **M0 — CAD dimensions.** Pull `hipOffset`, `L1`, `L2` from the Onshape
   model (blocking — see §7).
2. **M1 — Electrical smoke test.** ESP32-S3 + one PCA9685 + one leg (3
   servos) on a breadboard. Adapt `sesame-motor-tester.ino` to drive 3
   PCA9685 channels; verify full 0–180° range on all 3 physically before
   committing to any firmware architecture. (Cheap empirical check before
   building on an assumption, per standing practice.)
3. **M2 — IK verification in isolation.** Implement `solveLegIK` as a
   small standalone/unit-testable function; check it against known foot
   positions (e.g. straight-down stand position, full reach) before
   wiring it into the firmware loop.
4. **M3 — Port scaffolding.** Bring WiFi/web server/captive
   portal/face/serial-CLI over from sesame largely unmodified; wire in
   `NUM_SERVOS=15` and PCA9685 init in place of `ESP32Servo`.
5. **M4 — Single-leg pose test.** Drive one leg through IK-computed rest
   / stand / reach targets via the serial CLI.
6. **M5 — All 5 legs, stand pose.** Extend via the rotation-offset
   symmetry in §5.3.
7. **M6 — Wave gait walk cycle.** Implement and tune on real hardware;
   set final subtrim calibration.
8. **M7 — Full assembly + docs.** Final CAD parts, README/build-guide/
   wiring-guide rewritten for the 5-leg/PCA9685 layout.

## 7. Open questions / blocking items

- **CAD dimensions not yet available to this session.** I don't have
  fetch/API access to the linked Onshape document, so `hipOffset`, `L1`,
  `L2`, and the actual hip mounting angles are unknown — needed before §5
  IK constants can be finalized. Please pull these three measurements
  (or export a simple side-view sketch with dimensions) when convenient.
- **Servo torque.** sesame uses MG90S (~2.2 kg·cm). With 2 stacked
  servos per hip (yaw + pitch) and a heavier rock-textured body,
  hip-pitch in particular carries the whole leg's cantilever moment —
  worth a torque budget check once CAD mass/leg-length is known; may need
  a higher-torque servo than sesame's.
- **Does Rocky keep an OLED face?** Sesame's expressive-face concept is
  core to its identity; Project Hail Mary's Rocky doesn't have a screen
  face. Purely a design/identity call, not a blocker — face system ports
  over unmodified either way if kept.
- **Custom PCB vs off-the-shelf PCA9685 breakout for v1.** Recommend
  bringing up on an off-the-shelf PCA9685 breakout + the existing ESP32-S3
  board first (M1–M6), and only investing in a custom "Rocky Distro
  Board" PCB once the electrical/motion design is validated on hardware.
