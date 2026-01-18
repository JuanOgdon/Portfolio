



https://github.com/user-attachments/assets/11bee00c-bcda-4e80-bbcb-72c5a6000001



# Rocket Landing with a GA-Evolved Tiny Neural Network (Adaptive Curriculum + Anti-Exploit Fitness)

This project is a deep dive into *training engineering*: how to make a very small neural network learn a surprisingly complex control task by designing the fitness function, curriculum, evaluation procedure, and exploration strategy carefully.

The task is a simplified 2D rocket landing problem (planar motion + attitude): stabilize, maneuver toward a target, manage velocity, and perform a strict touchdown. The controller is evolved via a Genetic Algorithm (GA), and it is trained under **randomized initial conditions** so the resulting behavior must generalize rather than memorize a single trajectory.

---

## What I built

A complete training loop for evolving a minimal neural controller that can:
- recover attitude (keep the rocket upright),
- translate toward the landing pad (near the origin),
- reduce velocities in a controlled way *at the right time*,
- and satisfy a **strict landing definition** at touchdown.

The key theme here is that the “cleverness” lives in the training design:
- a fitness function that is interpretable, shaped, and resistant to reward hacks,
- a staged curriculum that gradually introduces complexity,
- adaptive difficulty scaling to prevent training collapse at stage boundaries,
- and GA techniques to avoid getting stuck in local optima and premature convergence.

---

## Controller concept 

The controller is a tiny feedforward neural network (MLP) that maps the rocket state to two actions:
- **throttle** (how much thrust to apply),
- **gimbal** (TVC direction / tilt command).

Inputs capture the essential dynamics (position, velocity, attitude, angular rate). The small architecture is a deliberate constraint: it forces the solution to come from good reward/curriculum design instead of brute-force capacity.

A side benefit: it’s very portfolio-friendly because the system is explainable and shows strong engineering decisions rather than “bigger model = better”.

---

## Core idea: a structured 200-point fitness (100 maneuvering + 100 landing)

I designed the fitness to be interpretable and to guide learning in the right order:

- **Maneuvering (0–100 points):** reward “good flight behavior” continuously across the episode.
- **Landing (+100 points):** reward only if strict touchdown criteria are met.

This creates a clear training message:
1) learn stable, controlled behavior in the air,
2) then prove you can stick the landing under strict constraints.

It also helps debug the training: if a policy gets ~80 maneuvering points but never lands, I know it’s behaving well but failing the final approach. If it gets low maneuvering points, it’s unstable or drifting.

---

## Maneuvering reward (0–100): shaped, gated, and intentionally hard to exploit

The maneuvering score is accumulated step-by-step and combines four components:

### 1) Uprightness (“attitude quality”)
This term rewards keeping:
- small angle (upright),
- small angular velocity (no spinning / oscillating).

This is the foundational behavior: if the rocket is tumbling, position and velocity rewards shouldn’t matter.

### 2) Position near the target (Gaussian around the origin)
This term rewards being near the landing pad using a Gaussian-shaped score centered at `(x, y) = (0, target_altitude)` or near ground depending on the phase of descent.

Crucial detail: **this term is gated by uprightness**.
- If the rocket is not sufficiently upright, it doesn’t get paid for drifting toward the target.
- This prevents learning “sideways sliding” or unstable approaches that still happen to move toward the origin.

### 3) Velocity control (Gaussian on vx, vy)
This rewards low velocity (especially near the landing region).

Crucial detail: **this term is gated by two conditions**:
- the rocket must be upright enough, and
- it must be close enough to the landing region.

This prevents a very common reward hack: “slow down far away, then fall uncontrolled later”. The reward structure enforces the natural order:
- stabilize attitude,
- move toward the target,
- only then aggressively kill velocity near the pad.

### 4) Fuel / excessive actuation penalty
A small penalty discourages wasting fuel or using high throttle constantly. This typically improves smoothness and reduces oscillatory “bang-bang-ish” behavior that might look energetic but is less stable.

### Why the gating matters (the real lesson)
Without gating, the GA often discovers shortcuts:
- being stable but drifting far away while farming uprightness,
- slowing down far from the pad,
- or oscillating thrust to accumulate shaping reward without converging to landing.

With gating, the reward becomes *state-dependent*: you only get paid for the “next” objective once the “previous” objective is reasonably satisfied. That’s one of the most important insights from this project.

---

## Landing bonus (+100): strict touchdown definition (deliberately unforgiving)

A landing is only counted as successful if multiple criteria are met simultaneously at contact, including:

- **angle**: must be close to upright,
- **angular velocity**: must be small (no rotation at touchdown),
- **x position**: must be close to pad center,
- **x velocity**: must be small (no lateral sliding),
- **y velocity**: must be small (soft descent),
- **y position/contact condition**: must represent real ground contact, not “almost touched”.

This strictness is intentional. It turns landing into a discrete achievement that can’t be faked by “almost good” behavior. In practice, this forces the policy to learn a stable final approach and not just a visually plausible descent.

---

## Anti-exploit mechanism: making landing strictly better than hovering

A common failure mode in shaped-reward control tasks is that the agent learns to hover near the ground to keep accumulating maneuvering reward rather than landing.

To eliminate that exploit, I implemented a simple but powerful trick:

- **When a successful landing occurs, the agent receives the +100 landing bonus**
- **plus the maximum maneuvering reward it *could have earned* for the remaining steps of the episode**

In other words: once you land, the episode becomes “as if you kept being perfect until the end” in terms of maneuvering points.

Why this works:
- If hovering near the ground could earn extra maneuvering points, it might beat landing.
- With this finish bonus, landing immediately dominates: you can’t do better by delaying touchdown.
- So the optimal strategy becomes: *land as soon as you can do it safely under the strict criteria*.

This one detail dramatically reduces reward hacking and makes training converge faster.

---

## Curriculum learning: 3 stages with explicit differences

The landing task has a classic difficulty pattern:
- learning attitude stabilization is already non-trivial,
- translating toward the target adds coupling,
- and the final seconds require precision and timing.

So I trained through three stages, each adding complexity and broadening generalization:

### Stage 1 — “Learn the concept of landing” (friendly initial conditions)
Goal: teach the policy the basic structure of “stabilize → descend → touch down”.

Typical characteristics:
- randomized horizontal offset (start not perfectly centered),
- relatively fixed or narrow-band altitude,
- minimal or no initial tilt,
- minimal or no initial velocities and angular rate.

Why: if you start with full randomness, the search space is huge and GA tends to converge to trivial local maxima (e.g., “don’t crash immediately” or “reduce spin a little”). Stage 1 gives the controller a learnable pathway.

### Stage 2 — “Recover attitude disturbances” (introduce tilt)
Goal: force the policy to learn correction dynamics, not just gentle descent.

Added difficulty:
- randomized initial attitude (tilt) around upright.

Effect:
- the policy must now allocate thrust and gimbal not only to slow descent but to recover orientation early enough to enable the later reward gates (position/velocity gates).

This stage often creates a new local maximum: “over-correcting tilt with aggressive thrust”, which looks like progress but prevents landing. Reward gating + curriculum helps escape that.

### Stage 3 — “Robustness and generalization” (full randomized starts)
Goal: make the policy robust across a wide variety of scenarios.

Added difficulty:
- wider horizontal offsets,
- randomized altitude (start higher or lower),
- randomized velocities (vx, vy),
- randomized angular rates (omega),
- overall broader distributions.

This stage is where the controller stops being a staged demo and becomes a general solution. It is also where local maxima are most common (e.g., “stabilize but never commit”, “hover and drift”, “hard brake too early then drop”).

---

## Adaptive difficulty scaling (what made stage transitions actually work)

Even with stages, a sudden jump in randomness can destroy performance. So instead of switching difficulty like a binary flag, I used an adaptive rule:

- Track **success streak** (consecutive successful landings).
- If the agent achieves **10 landings in a row** (with randomized starts),
- then increase the difficulty by a factor (e.g., +20% difficulty scaling).
- Repeat until reaching full difficulty for that stage, then move to the next stage.

Why 10 in a row matters:
- Because starts are randomized, the streak is a robustness test.
- A single “lucky landing” doesn’t advance difficulty; consistent performance does.

This mechanism effectively creates a smooth ramp from easy to hard, preventing the GA from repeatedly collapsing when the environment distribution shifts.

---

## GA training: escaping local maxima and maintaining exploration

Genetic algorithms are powerful for sparse-reward control, but they are vulnerable to premature convergence and local maxima. This project taught me how to *engineer* GA exploration so it keeps discovering better behaviors:

### Typical local maxima I ran into
- “Don’t crash immediately” (survive longer without true control).
- “Stabilize attitude only” (uprightness farming, no translation).
- “Approach the target but too fast” (fails strict landing criteria).
- “Hover forever near the pad” (reward shaping exploit, fixed by finish bonus).
- “Kill velocity too early far away” (fixed by velocity gating near the pad).

### How I overcame them
- **Reward gating**: prevents shortcut behaviors from scoring highly.
- **Finish bonus**: removes the hover exploit entirely.
- **Curriculum + adaptive difficulty**: provides a continuous learning path instead of a cliff.
- **Diversity injections (“immigrants”)**: reintroduces exploration when the population collapses to a mediocre strategy.
- **Multi-start evaluation**: forces robustness and prevents overfitting to a single initial condition.
- **Tuning exploration/exploitation**: adjusting mutation strength and diversity mechanisms to keep progress stable without randomizing away good solutions.

The main insight: the GA is only as good as the evaluation and reward design. Once those are engineered well, even a tiny network can learn complex control quickly and reliably.

---

## What I learned

- How to design reward shaping that is *aligned with the real objective*:
  - I learned that shaping terms must be gated and normalized, otherwise the agent optimizes the wrong thing (fake progress).
  - A good reward is not just “more terms”; it’s the correct structure and conditionality.

- How to eliminate reward exploits proactively:
  - The finish-bonus landing trick (landing gives +100 plus max remaining maneuvering points) made landing strictly optimal vs. hovering.
  - This is a general lesson: if the objective is terminal, ensure the reward landscape makes the terminal condition dominate.

- How to guide learning through “hard” solution spaces:
  - The landing task has multiple coupled requirements that are hard to learn simultaneously.
  - Staging (1→2→3) reduced the dimension of the search at each step.
  - Adaptive difficulty turned discrete stage jumps into a smooth ramp.

- How to handle local maxima systematically:
  - I learned to *identify* what local optimum the population is stuck in (attitude-only, hover, slow-down-far-away, etc.).
  - Then I modified either the fitness structure (gates/bonuses) or the training distribution (curriculum/adaptive difficulty) to reshape the landscape so the GA “sees” a better path.

- How to build robustness without brute force:
  - Randomized initial conditions and multi-start evaluation implicitly trained generalization.
  - Requiring a 10-landing streak before increasing difficulty made robustness part of the training objective, not an afterthought.

- How to balance exploration and exploitation in evolutionary learning:
  - Too little exploration → premature convergence to mediocre behavior.
  - Too much exploration → unstable progress and loss of good individuals.
  - I learned practical ways to control this (elitism + diversity injection + mutation tuning) to keep learning fast, stable, and effective.

- How to make small models do serious work:
  - Keeping the network tiny forced me to solve the real challenge: training design.
  - This is directly transferable to real GNC problems, where constraints and robustness matter more than raw model size.

---

## Media





https://github.com/user-attachments/assets/9390ee62-759d-4c4b-8b51-e77675bd125a


https://github.com/user-attachments/assets/5857608e-6910-49c2-8804-206e8f0c63e6



https://github.com/user-attachments/assets/bccf92a4-680f-49c7-8b5a-71e217fa9f4d


