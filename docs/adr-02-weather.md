# ADR-02: Weather model — Ornstein-Uhlenbeck process

**Status:** Accepted  
**Date:** 2025

---

## Context

Wind in the game must satisfy several constraints simultaneously:

1. **Stationarity** — wind speed should not drift to infinity over a long run.
2. **Temporal correlation** — consecutive wind values should be correlated so
   the platform experiences sustained gusts, not white noise.
3. **Seasonal character** — spring should feel calm, autumn should have
   sustained directional crosswind, winter should be violently unpredictable.
4. **Testability** — the statistical properties of the wind model should be
   analytically predictable so the unit tests can verify correct implementation
   without reference data.

---

## Decision

The wind vector evolves as a discrete Euler-Maruyama approximation of the
Ornstein-Uhlenbeck (OU) SDE:

    dX_t = θ(μ - X_t)dt + σ dW_t

which in discrete form becomes:

    x_{t+1} = x_t + θ(μ - x_t)Δt + σ√Δt · z,   z ~ N(0,1)

The stationary distribution is N(μ, σ²/2θ). Each season has its own (θ, σ,
μX, μY) tuple defined in the `SEASON_WIND` table. Autumn sets μX = 6 m/s
for sustained eastward crosswind; winter sets σ = 6.0 for high variance.

---

## Alternatives considered

**White noise (i.i.d. per frame)** — no temporal correlation, so the platform
oscillates rapidly rather than being pushed by sustained gusts. Gameplay is
frustrating and unrealistic; there is no meaningful strategic response.

**Deterministic periodic functions** — wind follows a predictable pattern that
a trained agent can memorise rather than generalise. Removes the stochastic
element that makes the environment interesting as an RL testbed.

**Fractional Brownian motion (fBm)** — captures long-range correlation more
realistically but has no closed-form stationary distribution, making it
impossible to write a unit test that checks the correct implementation. It also
requires storing a history of past values, increasing complexity.

**Lookup table from recorded meteorological data** — would require external
assets (violating the no-network-calls constraint) and breaks the seeded
reproducibility property.

---

## Consequences

The OU model's stationary variance σ²/(2θ) is analytically known, enabling the
unit test in `runTests.html` to verify the implementation with a 10 000-sample
convergence check (12% relative tolerance, burn-in 600 steps). The seasonal
parameter table cleanly separates the weather character of each season from the
simulation mechanics. The main limitation is that the OU process has no memory
beyond its current state, so multi-day weather patterns (storm fronts, calm
spells) require the overlay of deterministic gust events on top of the OU base
(implemented as spring gust impulses in `stepHazards()`).
