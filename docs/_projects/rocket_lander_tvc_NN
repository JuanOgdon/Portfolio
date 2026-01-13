# Rocket Landing with a GA-Evolved Tiny Neural Network (Adaptive Curriculum)

This project explores how far you can push a very small neural network controller when the training process is engineered carefully. A 2D rocket must perform controlled maneuvering and then achieve a strict touchdown, starting from randomized initial conditions so the policy must generalize rather than memorize a single trajectory.

> Replace these placeholders with your media:
>
> ![Training montage](assets/training_montage.gif)  
> ![Successful landing](assets/success_landing.gif)  
> ![Robustness test](assets/robustness_random_starts.gif)

---

## What I built

A minimal neural controller was evolved using a Genetic Algorithm (GA) to land a rocket in simulation. The focus was not “throw a bigger model at it”, but instead designing:
- a fitness function that cannot be easily exploited,
- a staged training process that gradually increases difficulty,
- and robustness mechanisms that force generalization across randomized starts.

The end goal was a controller that can repeatedly land under strict criteria, not just “look stable”.

---

## Core idea: a 200-point fitness that separates maneuvering from landing

I split performance into two clearly interpretable parts:
- 100 points for high-quality maneuvering (stable, centered, controlled speeds, efficient thrust usage)
- 100 points for a strict landing (touchdown only counts if multiple thresholds are simultaneously satisfied)

This keeps learning structured: first behave well in the air, then prove you can actually land.

---

## Maneuvering score (0–100): shaped, gated, and hard to game

The maneuvering reward is accumulated throughout the episode and built from four components:

1) Uprightness  
Encourages small angle and angular velocity. This is the foundation: if the rocket isn’t upright, nothing else matters.

2) Position near the target  
A Gaussian-shaped reward centered at the pad (near the origin). This term is gated: it only “pays” when the vehicle is sufficiently upright. That prevents sideways drifting while tilted from being rewarded.

3) Low velocity (especially near the pad)  
Another Gaussian-shaped reward on velocities. This is gated even more strictly: it only counts when the rocket is upright and already near the landing region. That prevents “reward hacking” where the agent learns to kill velocity far away and then falls back uncontrolled.

4) Fuel / actuation penalty  
A small penalty discourages excessive throttle usage. This helps avoid high-thrust oscillations that look energetic but are unstable or inefficient.

The key design lesson here: gating and normalization matter. Without gates, a policy can accumulate reward in unintended states (fake progress). With gates, the reward naturally enforces an order of operations:
stabilize attitude → move toward target → slow down near the pad.

---

## Landing score (+100): intentionally strict touchdown definition

The landing bonus is only granted if the rocket meets a set of simultaneous constraints at contact. Criteria include:

- angle small enough (upright at touchdown)
- angular velocity small enough (no spin at contact)
- x position within a tight band around the pad
- x velocity small enough (no lateral slide)
- y velocity small enough (soft descent rate)
- y position consistent with true ground contact (not “almost touching”)

A big emphasis in this project was strictness: near-misses are not counted as “pretty good”. This forces the controller to learn real stability and a controlled final approach.

---

## Anti-exploit detail: preventing “hovering beats landing”

A common failure mode with shaped rewards is learning to hover near the ground to farm points instead of committing to touchdown. To prevent that, I added logic that makes successfully landing more valuable than extending the episode for extra shaping reward. The result is that the best strategies converge toward decisive, controlled landings rather than indefinite hovering.

---

## Training strategy: 3 stages + adaptive difficulty scaling

Instead of training the hardest version from the start, I used a curriculum:

Stage 1 — basic descent and touchdown  
Starts are randomized in position but kept relatively “friendly” (minimal disturbances). The goal is to learn the landing concept.

Stage 2 — introduce attitude disturbance  
Adds randomized initial tilt so the controller must learn recovery plus descent control.

Stage 3 — full randomized starts  
Adds broader randomness: wider x offsets, varying altitudes, randomized velocities, and angular rate. This is where robustness is built.

### Adaptive difficulty (the turning point)
Even with stages, sometimes the jump to the next difficulty band was too large. So I implemented an adaptive rule:

- if the controller achieves 10 successful landings in a row,
- with randomized starting states,
- then difficulty increases by a chosen factor (e.g., +20%).

This “success streak → difficulty step” rule guided learning smoothly and avoided the classic brick-wall transition where training collapses after a stage change.

Because starts are randomized, “10 landings in a row” becomes a robustness filter, not luck.

---

## GA learning lessons: keeping evolution from stagnating

Genetic search is powerful, but it can stagnate. To keep exploration healthy I used standard but very effective techniques:
- elitism (preserve the best)
- mutation + crossover tuning
- injecting new random individuals (“immigrants”) to restore diversity
- evaluating performance across multiple randomized starts to avoid overfitting

The big takeaway: GA performance depends as much on evaluation design and diversity maintenance as on the neural network itself.

---

## What I learned (portfolio highlights)

- Reward shaping is engineering: gating, normalization, and anti-exploit logic can be the difference between real learning and fake progress.
- Strict success criteria matter: a hard landing definition produces genuinely stable controllers.
- Curriculum learning is not just a convenience; adaptive difficulty can be a critical mechanism to bridge learning gaps.
- Robustness can be trained implicitly by randomizing starts and evaluating across multiple initializations, even without explicitly injecting disturbances.
- Small networks can work surprisingly well when the training process is designed with care.

---

## Media

- Controlled descent + touchdown (best policy): `assets/success_landing.gif`
- Randomized start robustness montage: `assets/robustness_random_starts.gif`
- Stage progression comparison (Stage 1 → 2 → 3): `assets/stage_progression.gif`
