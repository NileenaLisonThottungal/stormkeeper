# ADR-01: Physics integrator — semi-implicit Euler

**Status:** Accepted  
**Date:** 2025

---

## Context

The simulation integrates two coupled differential equations per tick: the
vertical velocity of the platform (driven by net balloon lift, gravity, snow
load, and warm-gust force) and the angular velocity (driven by wind torque and
asymmetric load distribution). Both equations have explicit damping terms.

The integration runs at two different timesteps: PHYS\_DT = 1/60 s during
normal play, and TRAIN\_DT = 0.1 s during RL training (a 6× speedup needed to
reach 1 000 training episodes in browser-feasible time). The integrator must
remain stable at both.

---

## Decision

Semi-implicit Euler (also called symplectic Euler): the velocity update uses
the current forces (explicit), but the position update uses the *new* velocity
(implicit on position). For a spring-damper system of the form

    v' = f(x) - c·v
    x' = v

this reads:

    v_new = v + dt · f(x)
    x_new = x + dt · v_new

---

## Alternatives considered

**Forward (explicit) Euler** — simple but energy-divergent for spring-damper
systems. At TRAIN\_DT = 0.1 s and the damping constants used (PLAT\_VDAMP =
2.8, PLAT\_RDAMP = 3.5), forward Euler produces visible energy growth within
ten steps, making training results unreliable.

**RK4** — accurate to fourth order but requires four force evaluations per
step. At 60 Hz with particles, clouds, and six concurrent physics bodies this
is 4× the computational cost with no meaningful benefit: the dominant
integration error here is the discretisation of the OU wind process, not the
platform ODE.

**Velocity Verlet** — symplectic and second-order, but requires the force at
the next position, which couples the position and velocity updates. Adding
velocity-dependent damping (which the platform dynamics require) breaks the
clean Verlet structure and requires an implicit solve or a correction term.

---

## Consequences

Semi-implicit Euler is unconditionally stable for linear spring-damper systems
regardless of timestep, which is why TRAIN\_DT = 0.1 s produces physically
plausible trajectories without energy blow-up. The integrator is first-order
accurate, so it introduces O(dt) truncation error — at TRAIN\_DT this means
the agent trains on slightly different dynamics than it plays on. In practice
the gap is small relative to the OU wind noise, and the policy transfers
without re-tuning.
