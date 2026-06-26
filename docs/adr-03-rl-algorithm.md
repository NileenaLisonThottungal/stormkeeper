# ADR-03: RL algorithm — REINFORCE with Adam

**Status:** Accepted  
**Date:** 2025

---

## Context

The learning algorithm must run entirely in the browser in plain JavaScript with
no external libraries. It must be implementable from first principles in a
reasonable amount of code (~150 lines), understandable to an academic reviewer
reading the source, and capable of producing a policy that at least learns to
survive significantly longer than random. The action space has 32 discrete
actions; the observation space is a 47-dimensional real-valued vector.

---

## Decision

**REINFORCE** (Williams, 1992) — a Monte Carlo policy gradient algorithm.
After each complete episode, discounted returns G_t are computed back-to-front,
normalised across the episode as a simple variance-reduction baseline, then
used to weight the gradient of the log-probability of the selected action at
each step. The update rule is:

    θ ← θ + α · G_t · ∇_θ log π_θ(a_t | s_t)

The policy is a single hidden layer network: 47 → 32 tanh → 32 softmax.
Weights are initialised with Xavier uniform. The optimiser is **Adam** (lr =
3 × 10⁻⁴, β₁ = 0.9, β₂ = 0.999) rather than plain SGD, because the gradient
magnitudes across W1 and W2 differ by roughly one order of magnitude after the
tanh non-linearity, and Adam's per-parameter adaptive learning rate handles
this without manual tuning of separate layer-wise learning rates.

---

## Alternatives considered

**Tabular Q-learning** — requires discretising the 47-dimensional continuous
observation space. Even a coarse 3-value discretisation per dimension gives
3^47 ≈ 10^22 states, which is intractable. Dimensionality reduction (e.g.
hand-crafted features) would work but defeats the purpose of demonstrating a
learned representation.

**DQN** — addresses the Q-learning dimensionality problem via function
approximation but requires a replay buffer, a target network, and
ε-greedy exploration, which together add roughly 200 lines of infrastructure
code. For a single-file demonstration the complexity cost is high relative to
the benefit.

**Actor-Critic (A2C/A3C)** — lower variance than REINFORCE because the critic
provides a step-by-step advantage estimate rather than a Monte Carlo return.
However, it requires a second network (the value function), doubling the
weight count and the backpropagation code. With 32 hidden units the policy
network is already trainable with REINFORCE in browser time; the added
complexity of a critic is not justified.

**PPO** — state of the art for discrete action spaces but requires clipped
surrogate objectives, value loss, entropy bonus, and multiple epochs per batch.
This is approximately 300 additional lines and would make the algorithm code
the dominant part of the file rather than the environment.

---

## Consequences

REINFORCE is high-variance: gradient estimates from single trajectories are
noisy, and convergence is slow relative to actor-critic or PPO methods. The
normalised-return baseline reduces variance but is weaker than a learned value
baseline because it normalises within an episode rather than across a rolling
mean of episode returns. In practice the agent learns to keep balloons aloft
(the dense survival signal) within ~200 episodes and largely ignores plant
harvesting (the sparse signal). This is a known limitation and is documented
honestly in the reward shaping comments in the source. A production setup would
use PPO with a value network in Python/JAX and run for 10 000+ episodes;
the JS REINFORCE implementation here is intended to be readable and
self-contained, not to maximise return.

**Honest performance note:** the agent does not reach human-level performance.
It reliably survives spring and sometimes reaches summer; it rarely survives
autumn. The primary bottleneck is the sparse harvest reward combined with the
high variance of Monte Carlo returns — problems that better algorithms address
but that are instructive to observe precisely because they are left unsolved.
