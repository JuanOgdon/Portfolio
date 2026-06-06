




https://github.com/user-attachments/assets/a3ba9928-0788-481b-a9e2-bfbf783275e5




# Rocket Landing with a GA-Evolved Tiny Neural Network (Adaptive Curriculum + Anti-Exploit Fitness)

This project is a deep dive into *training engineering*: how to make a very small neural network learn a surprisingly complex control task by designing the fitness function, curriculum, evaluation procedure, and exploration strategy carefully.

The task is a simplified 2D rocket landing problem (planar motion + attitude): stabilize, maneuver toward a target, manage velocity, and execute a strict suicide-burn touchdown. The controller is evolved via a Genetic Algorithm (GA) under **randomized initial conditions**, so the resulting behavior must generalize rather than memorize a single trajectory.

The paper has been published. Three physically distinct rocket configurations — Viper (baseline), Colossus (heavy), and Dart (agile) — are each trained and validated under the same framework.

---

## Where it started: the inverted pendulum

The direct predecessor of this project was a classic control problem: balance an inverted pendulum using a GA. Small network, physics simulation, fitness measuring how long the pole stayed upright. It worked, and it taught me the core loop — GA framework, fitness design, curriculum thinking — but the problem is too forgiving. A pole either balances or it doesn't; the failure modes are obvious and the reward landscape is smooth.

Rocket landing is harder in every dimension that matters. The task has a terminal condition, the physics couples attitude and position nonlinearly, the burn timing is critical, and the GA can find dozens of ways to appear to make progress while learning nothing useful. The transition from pendulum to rocket was the transition from "GA as a toy" to "GA as a research tool."

---

## What I built

A complete training loop for evolving a minimal neural controller that can:

- recover attitude (keep the rocket upright),
- translate toward the landing pad (near the origin),
- reduce velocities in a controlled way *at the right time*,
- and satisfy a **strict landing definition** at touchdown.

The key theme: the "cleverness" lives in the training design — a fitness function that is interpretable, shaped, and resistant to reward hacks; a staged curriculum that gradually introduces complexity; adaptive difficulty scaling to prevent collapse at stage boundaries; and GA techniques to avoid premature convergence.

---

## Controller architecture

The controller is a tiny feedforward MLP (122 total weights): **9 inputs → 10 hidden neurons (ReLU) → 2 outputs** (throttle and gimbal). Outputs are filtered through first-order slew-rate limiters, so the network has to plan ahead — it can't change the engine state instantaneously. The actual throttle and gimbal commands are fed back as inputs, giving the network full observability of its own actuator state without needing memory or recurrent connections.

The small architecture is a deliberate constraint. It forces the solution to come from good reward and curriculum design rather than brute-force model capacity. It also makes the system explainable: a 122-weight network that lands a rocket is a statement about the training system, not the network.

### The input that made the biggest difference

Eight of the nine inputs are standard: position (x, y), velocity (vx, vy), attitude and angular rate (θ, ω), and the current throttle and gimbal commands. The ninth required more thought.

`delta_burn` is a physics-derived signal:

```
delta_burn = (y − vy² / (2 × a_net)) / 50
```

It computes the difference between the rocket's current altitude and the altitude where it *should* ignite for an optimal suicide burn — the point where, if the engine fires at full thrust right now, it will just reach zero velocity at ground level. When `delta_burn` is positive, the rocket is still too high to commit. When it crosses zero, it's time.

Without this signal, the GA reliably discovers the simplest valid strategy: fire the engine immediately from episode start and decelerate throughout. This "always-on" behavior works for easy initial conditions but wastes fuel and doesn't generalize. With `delta_burn` in the input, the network has direct access to the correct timing decision instead of having to infer it from position and velocity.

All inputs use fixed physical scales, not episode-relative normalization. A rocket starting at 20m and one starting at 50m see the same representation of their true state. Episode-relative normalization would destroy the meaning of `delta_burn`.

---

## Core idea: a structured fitness (maneuvering score + landing bonus)

I designed the fitness to be interpretable and to guide learning in the right order:

- **Maneuvering (0–~150 points):** reward "good flight behavior" continuously across the episode.
- **Landing bonus:** reward only if strict touchdown criteria are met — plus a finish bonus for time remaining.

This creates a clear training message: learn stable, controlled behavior in the air, then prove you can stick the landing under strict constraints. It also helps debug training: if a policy gets high maneuvering points but never lands, it's behaving well but failing the final approach. If maneuvering points are low, it's unstable or drifting.

---

## Maneuvering reward: shaped, gated, and intentionally hard to exploit

The maneuvering score accumulates step-by-step and combines four components:

### 1) Uprightness

Rewards small angle and small angular velocity. This is the foundational behavior: if the rocket is tumbling, nothing else should matter.

### 2) Position near the target (height-fair)

Rewards proximity to the landing pad using a Gaussian-shaped score. 

**Crucial detail 1: gated by uprightness.** If the rocket isn't sufficiently upright, it doesn't get paid for drifting toward the target. This prevents "sideways sliding" or unstable approaches that happen to move toward the origin.

**Crucial detail 2: height-fair.** The original design used `G(x, σ_x) × G(y, σ_y)` — a Gaussian on altitude as well. The problem: a rocket starting at y=50m earns near-zero position reward for most of its descent, while one starting at y=20m begins earning meaningful reward much earlier, *even if both controllers are equally competent*. This biased mean fitness toward low-altitude rollouts and stalled curriculum advancement. The fix: replace the altitude Gaussian with an altitude progress fraction `(1 − y/y_start)`. Same reward signal regardless of starting height, same proportional descent progress.

### 3) Velocity control

Rewards low velocity, especially near the landing region.

**Crucial detail: gated by two conditions** — the rocket must be upright *and* close enough to the pad. This prevents a very common exploit: "slow down far away, then fall uncontrolled later." The structure enforces the natural order: stabilize → navigate → kill velocity near the pad.

### 4) Fuel / actuation penalty

A per-step penalty discourages sustained low-throttle burning. The formula matters: `w_fuel × (1 − throttle_norm)²`, where `throttle_norm` is 0 at minimum throttle and 1 at full throttle. This costs exactly **zero at full throttle** and increases the less committed the burn. The original formula penalized full throttle more than minimum throttle — the opposite of what was intended, and it produced controllers that avoided committing to full burns. Inverting the formula was a one-line change that shifted the evolved behavior noticeably.

### Why the gating matters

Without gating, the GA reliably discovers shortcuts:
- farming uprightness reward while drifting far from the pad,
- slowing down far away from the landing zone,
- oscillating thrust to accumulate shaping reward without converging.

With gating, reward becomes *state-dependent*: you only get paid for the next objective once the previous one is satisfied. That conditional structure is one of the most important insights from this project.

---

## Landing bonus: strict touchdown + anti-exploit finish bonus

A landing is counted as successful only if multiple criteria are met simultaneously at contact: near-upright angle, small angular velocity, near the pad center, small lateral velocity, soft descent velocity, and real ground contact.

This strictness is intentional. It turns landing into a discrete achievement that can't be faked by "almost good" behavior.

### Making landing strictly better than hovering

A common failure mode in shaped-reward tasks: the agent learns to hover near the ground to keep accumulating maneuvering reward rather than landing. It's a locally stable strategy that looks like progress.

To close this exploit completely:

- **On a successful landing, the agent receives the landing bonus plus the maximum maneuvering reward it *could have earned* for all remaining episode steps.**

Landing immediately dominates: you can't do better by delaying touchdown. The optimal strategy becomes *land as soon as you can satisfy the strict criteria*.

A second mechanism reinforces this: `landing_factor` scales the finish bonus by 1.75 in early stages and 2.5 in later stages, making earlier landings worth more than later ones. The two-sided pressure — landing early earns more *and* hovering costs more via the fuel penalty — eliminates the hover strategy without any explicit constraint.

---

## Hard constraints vs. soft selection pressure (a lesson learned the hard way)

Stage 4 was originally designed to require single-burn landings using a hard constraint: if the engine ever shut off and re-ignited, it locked permanently. The entire Stage 4 starting population — graduates from Stage 3, all trained with multi-burst control — fired the engine, modulated throttle below the re-ignition threshold, and crashed, scoring ~15 pts. Non-firers, which simply never ignited, accumulated ~70 pts of shaping reward.

Since non-firers scored higher than crashed multi-burst controllers, the GA sorted non-firers to the top of the population in the first generation. With mutation strength 0.07, offspring of non-firers stayed non-firers. The population converged to "never fire" within the first few generations and stayed there for 4000 generations. Zero landings.

The fix replaced the hard constraint with a penalty: `w_reignition × n_reignitions` added to fitness. Multi-burst landings still score ~200 pts — well above non-firers at ~70 pts — so the GA can bootstrap from Stage 3 graduates. Single-burn landings score ~300 pts and dominate selection. The graduation threshold is set so only single-burn controllers can pass, maintaining the behavioral objective. The implementation is completely different; the result is identical.

**General lesson**: hard behavioral constraints that score the constrained alternative worse than "do nothing" create inescapable fitness traps. Behavioral constraints should be enforced through relative fitness differences, not absolute lockouts.

---

## Curriculum learning: 4 stages with explicit differences

<img width="970" height="635" alt="IC" src="https://github.com/user-attachments/assets/88a9fdb1-33b4-417f-a647-efd8250497ee" />


### Stage 1 — "Learn the concept of landing"

Goal: teach the basic structure of "stabilize → descend → touch down" from friendly initial conditions (small horizontal offset, upright start, minimal initial velocities). Discovering the first landing at all requires a large population (1000 individuals across 4 islands) and high mutation strength. Once a working strategy appears, the population shrinks to 200 and mutation drops — concentrating the most expensive search budget exactly where it is needed.

### Stage 2 — "Recover attitude disturbances"

Introduces randomized initial tilt. The controller must now allocate thrust and gimbal not only to slow descent but to recover orientation early enough to activate the position and velocity reward gates. This stage typically creates a new local maximum: over-correcting tilt with aggressive thrust, which looks like progress but prevents landing. Reward gating and the `w_reignition` penalty (light, at this stage) help escape it.

### Stage 3 — "Robustness and generalization"

Full randomized starts: wider horizontal offsets, randomized altitude, velocities, angular rates. The burn timing pressure (`w_early_burn` penalty for engine-on above optimal ignition altitude) is introduced here, progressively ramping to push the evolved strategy toward the correct suicide-burn timing.

This is where IC narrowing is most tempting — tightening the distribution to force precision. Empirically, it causes catastrophic forgetting: conditions that appeared frequently in Stage 3 become rare events in Stage 4, and the policy forgets how to handle them within ~300 generations. The fix: keep Stage 4's IC distribution identical to Stage 3. **Robustness must come from the IC distribution. Precision must come from the success criterion. Never confuse the two.**

### Stage 4 — "Precision and commitment"

Same IC spread as Stage 3, but tighter landing velocity thresholds, higher evaluation rollouts (10), and maximum penalty pressure for early burns and re-ignitions. The `lander_only_parents` flag restricts breeding to individuals that have successfully landed at least once — once landers exist in the population, non-landers stop contributing to the next generation.

---

## Adaptive difficulty scaling

Even with stages, a sudden jump in randomness can destroy performance. Instead of switching difficulty as a binary flag, each stage uses a difficulty parameter `d ∈ [0, 1]` that scales the IC spread:

- Track a **success streak** (consecutive generations meeting the landing threshold).
- Once 8–15 consecutive generations pass (depending on stage), advance `d` by 0.2.
- Repeat until `d = 1.0`, then advance to the next stage.

This creates a smooth ramp from easy to hard, preventing the GA from collapsing when the environment distribution shifts. A streak of 10 with randomized starts is a robustness test, not a single lucky trial.

---

## GA structure: 4 islands, avoiding premature convergence

A 122-weight network is easy to overfit to a small parent pool. Diversity collapses within tens of generations in a single population.

The island model runs **4 independent subpopulations** that evolve in parallel and periodically exchange their best individuals (every 8 generations). Islands independently discover qualitatively different strategies; migration lets useful elements from distinct lineages eventually recombine.

A subtle bug in the original migration code: all migrants were assigned to the same slot (`pop[-1]`), so only one migrant survived per migration event. The fix assigns each migrant from a distinct island to a distinct target slot, so 3 migrants per event actually persist.

Additional diversity mechanisms: a high initial immigration rate (20% random-weight individuals for the first 30 generations, decaying to baseline) ensures the discovery phase doesn't converge prematurely. Neuron-wise crossover treats each hidden neuron's full weight vector as an atomic unit, preserving functional sub-circuits across generations rather than mixing individual weights incoherently.

---

## Problems I ran into (selected)

**Bimodal fitness and mean thresholds.** When the population mixes landers (~300 pts) and non-landers (~70 pts), the mean fitness is dominated by non-landers. A mean-fitness graduation threshold doesn't reliably distinguish "3/10 landings" from "5/10 landings." The fix was a streak criterion based on a count threshold with enough evaluation rollouts that lucky streaks are statistically rare. Getting the number of rollouts wrong (too few) lets a specialist that only works from easy initial conditions graduate by chance, producing a controller that fails on the majority of the training distribution.

**Fitness threshold at the expected mean.** Setting the graduation threshold exactly at the expected mean of a "just-passing" agent means roughly 50% per-generation pass rate, making a long streak geometrically unlikely regardless of agent quality — the agent needs to be consistently above average, not just average. The fix is to include variance slack: set the threshold 15 pts below the expected mean of a clearly-passing agent.

**Weight initialization and output saturation.** With uniform random initialization (scale ~1.0) and input values like y ≈ 20, the W1 matrix generates pre-activation values with std ≈ 8 per neuron, pushing all ReLU outputs to their bounds. The entire Stage 1 discovery phase starts with populations where every network commands maximum throttle and maximum gimbal — all crash identically and provide zero signal for selection. He/Xavier initialization (scale = `sqrt(2/fan_in) ≈ 0.47`) combined with tanh-scale input normalization reduces pre-activation std to ~0.18, restoring diverse outputs from generation 1.

---

## Three rocket families

The same architecture and training framework generalizes across physically distinct vehicles:

| Rocket | Mass | TWR | Character |
|--------|------|-----|-----------|
| **Viper** | 1.0 kg | 1.5 | Balanced baseline |
| **Colossus** | 5.0 kg | 1.3 | Heavy, sluggish — barely exceeds gravity |
| **Dart** | 0.5 kg | 2.5 | Ultra-agile, wide throttle range |

Each required adjusting the penalty design, not just the physical parameters. Colossus, with only 30% net upward acceleration, must ignite earlier and burn longer than Viper. Applying the same `w_early_burn` penalty (which penalizes any engine-on time above the optimal altitude, regardless of throttle level) directly punished Colossus's physically correct full-throttle behavior. After 3200 stagnant generations on Stage 3, the diagnosis was clear: the penalty was fighting the physics. It was disabled entirely for Colossus's Stage 3; fuel efficiency was incentivized through the throttle-commitment formula instead. The `delta_burn` signal adapts automatically — it computes optimal ignition altitude from each rocket's actual thrust parameters — so the network receives a physically correct timing signal across all three configurations with no architecture changes.

https://github.com/user-attachments/assets/bcf2f265-ddb4-4152-b1ea-dce664f0a133


https://github.com/user-attachments/assets/22044800-dbf8-4f15-b8ff-45dc2f9fe1e0


https://github.com/user-attachments/assets/c18a2e53-419c-448c-bba7-012990806b32


---

## What I learned

- **Good reward design is conditioned, not just weighted.** Gating terms by state prevents shortcut strategies. A reward that fires unconditionally is an exploit waiting to be found. Next you will see a video for the hover exploit. I leave you as a thought experiment to think what could have caused this and how to solve it (Ask me about it if you are interested).



https://github.com/user-attachments/assets/d55d011f-4a29-4ea3-9b9b-7779144f33cd





- **Terminal objectives need terminal dominance.** If the goal is landing, landing must score more than any alternative behavior in every possible state. The finish bonus (landing pays out all remaining steps) is the mechanism; without it, hovering is locally rational.

- **Hard constraints that score the alternative worse than "do nothing" are inescapable traps.** This is the clearest lesson from the `one_shot_ignition` failure. Soft selection pressure achieves the same behavioral objective without creating a "never fire" attractor.

- **Robustness and precision are controlled by different knobs.** IC spread sets what situations the controller has seen; success criteria set what counts as passing. Narrowing the IC spread to improve precision destroys robustness. The two must be adjusted independently.

- **The evaluation procedure is part of the training system.** Too few evaluation rollouts, and specialists graduate. Thresholds set at the expected mean of a passing agent make long streaks geometrically impossible. The number of rollouts must scale with IC spread and the bimodality of the fitness landscape.

- **Small models can do serious work — but only with seriously designed training systems.** The 122-weight network lands a rocket in three different physical configurations because reward gating, curriculum staging, island diversity, behavioral ramps, physics-derived inputs, and proper initialization each do real work. Remove any one of them and the failure mode is specific and traceable.

---

## Other

https://github.com/user-attachments/assets/f06ba7e0-d101-48e3-8a98-fe1bd8e68f70

 Media

https://github.com/JuanOgdon/Portfolio/assets/201483014/9390ee62-759d-4c4b-8b51-e77675bd125a

https://github.com/JuanOgdon/Portfolio/assets/201483014/5857608e-6910-49c2-8804-206e8f0c63e6

https://github.com/JuanOgdon/Portfolio/assets/201483014/bccf92a4-680f-49c7-8b5a-71e217fa9f4d
