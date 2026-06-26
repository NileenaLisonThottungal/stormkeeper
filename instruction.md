# Stormkeeper — build instructions

You are building a single-file browser game called **Stormkeeper** that doubles
as a reinforcement-learning testbed. This is a portfolio project for a master's
application in AI. Optimize for technical depth, honesty in the README, and
interview-defensibility — not for speed of delivery.

Work iteratively. Do not try to produce the whole thing in one pass. Follow the
stages below in order. At the end of each stage, stop, run the game, and
self-critique against the acceptance criteria before moving on.

---

## Project shape

- One repository, MIT licensed.
- Deliverable is a single `index.html` that runs by opening it in a browser.
  No build step, no frameworks, no network calls, no external assets.
- Vanilla JavaScript + Canvas 2D. No TypeScript, no bundler, no npm.
- A `README.md` written for a German AI faculty reviewer, not for a recruiter.
- A `tests/` folder with a tiny hand-rolled assertion runner (`runTests.html`)
  that exercises the pure functions: RNG determinism, OU process statistics,
  spatial hash correctness, physics integrator stability. No test frameworks.
- A `docs/` folder with three short ADRs (architecture decision records):
  one for the physics integrator choice, one for the weather model, one for
  the RL algorithm choice. Each ADR is ~200 words: context, decision,
  alternatives considered, consequences.

---

## The concept

The player keeps a small floating sky-garden aloft through a full year of
weather. Balloons provide lift and decay over time. Each season brings
distinct hazards. Plants grow on the platform and score when harvested.
The further into the year survived, the higher the score.

Tone: atmospheric, painterly, a little melancholic. Not arcade. Not cartoon.

---

## Visual quality bar — read this carefully

The user asked for "hyper-realistic UI." In a 2D canvas game with no external
assets, hyper-realism means **painterly realism, not photorealism**. Photoreal
rasters are impossible without assets. What is achievable and what you must
deliver:

- **Layered parallax skies** rendered procedurally: 4-6 depth layers, each a
  gradient with subtle noise, drifting at different speeds. The horizon shifts
  color through the day-night cycle within each season.
- **Volumetric-looking clouds** built from stacked semi-transparent metaballs
  with simulated edge softness (multi-pass radial gradients, not hard circles).
  Clouds deform slowly using Perlin or simplex noise on their control points.
- **Believable balloons**: each balloon is rendered with a radial gradient for
  body shading, a specular highlight offset toward the light source, a
  rendered string with catenary sag, and a subtle shadow on the platform when
  directly above it. Balloons sway with a damped pendulum motion off their
  tether point.
- **Particle hazards with depth**: snowflakes, raindrops, and leaves render at
  varied sizes and opacities to suggest distance. Closer particles motion-blur
  slightly along their velocity vector (short alpha-faded line, not a sprite).
  Snowflakes are six-fold symmetric, generated procedurally per particle and
  cached.
- **A platform that reads as a real object**: wood-grain rendered with layered
  noise, edge bevels with two-tone shading, plants that cast soft shadows
  matching the current sun angle, snow accumulation that follows the
  platform's actual tilt and forms a realistic angle-of-repose pile.
- **Lightning** rendered as a recursive jagged path with bright core, falloff
  halo, and a screen-wide flash frame on strike. Telegraph reticle is a slowly
  contracting circle with crosshair, not a red square.
- **Seasonal palette transitions** that cross-fade every frame over ~5 seconds,
  not snap. The sky, the platform tone, the plant colors, and the particle
  hue all shift together.
- **Subtle camera life**: the camera breathes with a 0.5px low-frequency sway
  and reacts to strong wind gusts with a damped offset.
- **HUD typography**: serif (Georgia or system serif) for title, sans for
  numeric readouts, all anti-aliased and rendered at integer pixel positions
  to avoid blur. Muted off-white text on a thin translucent panel, not bright
  white on a solid bar.

What is forbidden because it betrays the realism goal:
- Emoji, cartoon faces, hard-edged primary-color shapes.
- CSS box-shadows or filters on canvas elements (faked depth).
- Hard `fillRect`-style HUD bars without inner shading and border.
- Snapping seasonal transitions.
- Uniform-size, uniform-opacity particle fields.

If you can't make something look right with code, make it smaller and more
restrained rather than bigger and obvious. Restraint reads as intentional.

---

## Stage 1 — Skeleton and rendering pipeline

Build the rendering and time loop first, with no gameplay. Acceptance:
- Canvas fills the window and resizes correctly.
- Fixed-timestep physics at 60 Hz with interpolated rendering. Document the
  integration scheme (semi-implicit Euler) in a comment.
- A test scene shows: layered sky, drifting procedural clouds, the platform
  with three balloons holding it up, the four plant slots empty.
- Day-night cycle visible over a 60-second loop (for testing — real cycle
  will be longer).
- Runs at a stable 60 FPS with the dev tools profiler open.

Stop here. Open it. Confirm it looks like the visual bar above before moving on.

---

## Stage 2 — Physics and weather

Add the simulation layer. Acceptance:
- Balloons: buoyancy, mass, lift that decays. Lift below weight = balloon
  sinks and detaches. All constants named with units in comments.
- Wind: 2D vector evolving via an Ornstein-Uhlenbeck process. Parameters
  (theta, sigma, mu) per season, defined at top of file. The wind unit test
  must confirm the empirical mean and variance match theoretical values
  within tolerance over 10k samples.
- Hazard particles: real per-frame integration with mass, drag, drift.
  Spatial hash grid for collision queries. Unit test confirms the spatial
  hash returns the same neighbors as a brute-force nested loop on a seeded
  random scene.
- Deterministic seeded RNG (mulberry32). Seed shown in HUD. Unit test
  confirms identical seed → identical state after N steps.
- Platform tilt computed from real weight distribution, not faked.

Stop. Run tests. Confirm physics feels right (not floaty, not stiff) before
moving on.

---

## Stage 3 — Gameplay and seasons

Add the player interactions, seasonal weather phenomena, plants, scoring,
and the difficulty curve described below. Acceptance:
- Three actions work: tap balloon to re-inflate (with per-balloon cooldown),
  click-and-drag to warm-gust (stamina-limited), tap ripe plant to harvest.
- Four seasons with distinct phenomena:
  - Spring: gentle rain, sudden gusts.
  - Summer: heat shimmer reducing lift, telegraphed lightning.
  - Autumn: falling leaves accumulating, sustained crosswindswhat is the gams
  - Winter: snow accumulation, ice on balloons, blizzards.
- Plants: 4 slots, season-dependent yields (sunflower 2x in summer,
  frost-lily winter-only, etc.). Define plants in a config table.
- Difficulty tuned so an average run dies in spring or summer (60-90s) and
  a skilled run clears winter (6-8 min).
- End card on garden fall: season reached, days survived, score, seed,
  "begin again."

Do a balance pass at the end of this stage. Document tuning constants and
trade-offs in a comment block.

---

## Stage 4 — RL environment and agent

This is the stage that makes the project a master's-application artifact.
Do not skip or shortcut it. Acceptance:

- A clean Gym-style env interface in the same file:
  `env.reset(seed) → observation`
  `env.step(action) → {observation, reward, done, info}`
- Observation vector (fixed length): wind vector (2), balloon lifts (max 6,
  zero-padded), platform tilt (1), snow load (1), stamina (1), one-hot
  season (4), nearest-k hazard positions and velocities (k=8, padded).
- Discrete action space: {do_nothing, tap_balloon_i, gust_at_grid_xy,
  harvest_plant_i}. The grid for gust is 6x4 to keep the action space
  manageable.
- Reward shaping: small positive per surviving frame, larger positive on
  harvest, large negative on balloon loss or tip-over. Document in a
  comment what reward hacks this shaping invites (e.g. agent might
  ignore late-game plants for stable early survival).
- A from-scratch learning algorithm in plain JS:
  - Default to **REINFORCE with a single-hidden-layer policy network**
    (32 units, tanh, softmax output). Numpy-style matrix ops written by
    hand. No ML libraries. Adam optimizer or plain SGD with momentum —
    your choice, justify in ADR.
  - Tabular Q-learning only if you can argue the discretization works.
    Document the choice in the ADR.
- An "Agent" button in the HUD toggles between three modes:
  Human / Training / Playback.
- In Training mode, the renderer is skipped and a larger physics timestep
  is acceptable (document the shortcut). A small line chart in the corner
  shows mean episode return over the last 100 episodes. Aim for 1000
  episodes in 1-3 minutes on a mid-range laptop.
- In Playback mode, the trained policy plays a single episode at normal
  speed with full rendering. Trained weights are dumped to a JS variable
  the user can copy and paste back to skip retraining.
- Be honest in comments: if the agent only learns to keep balloons aloft
  and ignores harvesting, say so. If returns plateau below human level,
  say so. Reviewers value honesty over claims.

---

## Stage 5 — Documentation

After the code works, write:

- **README.md**: framed as "Stormkeeper: a reproducible stochastic
  environment and from-scratch policy-gradient agent." Lead with what the
  project demonstrates (OU process, semi-implicit Euler integrator,
  spatial hashing, mulberry32 RNG, Gym-style env interface, REINFORCE
  from scratch). Include a screenshot, a return-curve image, and a
  "limitations and what a production setup would do differently" section.
  Do not market it as a game in the README — frame it as a testbed.
- **Three ADRs** in `docs/` as described in "Project shape."
- A 150-word **CV bullet** at the bottom of the README, ready to copy:
  emphasize the environment design, the from-scratch agent, and the
  reproducibility, not the visual polish.
- A short **"how to read this codebase"** section pointing reviewers at
  the 5-6 functions worth reading first.

---

## Working style for this build

- After each stage: stop, run, screenshot, self-critique against the
  acceptance criteria. Write the critique in your response before moving on.
- If something doesn't work, do not paper over it. Say what's broken, why,
  and what you'd do with more time.
- Do not invent benchmarks, performance numbers, or claimed agent
  performance. Measure or omit.
- Constants over magic numbers, always with a unit comment.
- The code is read by a reviewer. Comments should explain *why*, not *what*.