# Rocky Robot — Side Findings & Future Ideas

Captured while researching sesame-robot and drafting `rocky-robot-design.md`.
Not part of the core v1 plan — revisit opportunistically.

- [ ] **Onshape CAD not machine-accessible from this session.** Could not
      fetch `hipOffset`/`L1`/`L2` dimensions from the linked document.
      This is the single blocking item for finalizing the IK constants in
      the design plan (§7) — needs manual export from Onshape.
- [ ] **sesame-robot's own firmware README already flags the PWM-timer
      scaling risk** (`ESP32PWM::allocateTimer(0..3)`, only 4 hardware
      timers, called out as a concern at just 8 servos) — this independently
      confirms the PCA9685 decision was right, not just a hunch.
- [ ] **`sesame_studio.py` (software/sesame-studio/) is hardcoded to
      exactly 8 servos / 4-leg schematic** with per-servo-group angle
      validation baked into `add_frame()`. If a Rocky pose-composer tool
      is built later (`rocky-studio`, mentioned as v2 in the design doc),
      it's worth generalizing this into a data-driven tool (reads a
      leg/joint config) rather than writing a second hardcoded one —
      would benefit sesame-robot too if ever contributed upstream.
- [ ] **Rocky (the book character) uses some limbs for fine manipulation**,
      not just walking. Long-shot future idea: could 1-2 of the 5 legs be
      designed/driven as dual-purpose walk+manipulate limbs? Not in scope
      for v1 (adds an entirely different control problem — grasping,
      different end-effector), but worth a passing thought later.
- [ ] **IMU/gyro for balance feedback.** sesame's README explicitly lists
      "Sensor integration (Ultrasonic, Gyro, etc.)" as an open contribution
      area it never built. A wave gait with 5 legs is inherently more
      stable than sesame's shuffle, but a cheap IMU could still help detect
      tips/stumbles on uneven surfaces later — v2+ idea, not blocking.
- [ ] **sesame-robot's own repo hygiene note (unrelated to Rocky):** the
      `software/sesame-studio/.ipynb_checkpoints/` folder in this repo
      contains stray Jupyter checkpoint clutter (duplicate old README/script
      copies) and an untracked `Untitled.ipynb` was added in the "adapted
      to own hardware" commit — probably accidental, worth cleaning up in
      sesame-robot itself at some point (separate from Rocky work).
