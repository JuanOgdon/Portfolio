---
title: "Inverted Pendulum — Genetic Algorithm + Neural Network Control"
date: 2025-09-24
tags: [controls, RL, genetic-algorithm, python, simulation, manim]
status: in-progress
summary: "I trained a small neural network with a genetic algorithm to balance an inverted pendulum, with exploration/robustness tricks that transfer to TVC-style rockets."
links:
  repo: ""
  video: ""
---

## Introduction
The **inverted pendulum on a cart** is a classic underactuated control problem: the pole must be kept upright while the cart moves laterally on a rail. It’s a great proxy for **thrust vector control (TVC) rockets**: both are unstable, have fast angular dynamics, and require attitude stabilization with limited actuation. While a TVC rocket actuates **thrust direction** to generate a restoring moment, the cart actuates **horizontal force**—but the **feedback structure** (angle/ang. rate ↔ lateral motion ↔ restoring action) is strikingly similar.

This project explores a **learning-based controller**: a small **neural network** whose weights are evolved with a **genetic algorithm (GA)**. The controller observes state and outputs a force command. The GA’s fitness balances “stay upright fast” with “stay smooth and centered.”

## State, Action, Termination
- **Inputs (state):**  
  - `θ` (angle, upright = 0)  
  - `θ̇` (angular rate)  
  - `x` (cart position)  
  - `ẋ` (cart velocity)  
  - **“is-in-zone”** (binary step = 1 if the cart is within a center band, e.g. \|x\| ≤ 1.5 m).  
    This helps the network learn where it should **stabilize**.
- **Output (action):** **Horizontal force** `F` on the cart.  
  > I chose **force** (not velocity) to keep the control signal physically grounded for a future real rig.
- **Episode end:** if the cart hits the rail limit (e.g., \|x\| ≥ 5 m) or time expires.

## Controller Architecture (NN + GA)
- **Neural net:** lightweight MLP (inputs → hidden → 1 output).  
- **Genetic Algorithm:** population of candidate weight vectors (individuals), mutation, selection, elitism, and **immigrants** for exploration.

## Fitness (Scoring) Design
I designed the score to **strongly prefer true stabilization** over “lucky spins”:

1. **Exponential points for consecutive upright time**  
   Each time step `Δt` spent with small \|θ\| increases the score **exponentially**, rewarding genuinely balanced behavior over controllers that happen to pass near upright occasionally.
2. **Multiplicative shaping terms**  
   The base score is **multiplied** by:
   - **Time-decay factor**: favors solutions that reach upright **faster** (earlier time → higher multiplier).
   - **Low-velocity multipliers** for `ẋ` and `θ̇`: extra credit for **smoothness** and reduced oscillations/jitter.
   - **Centering bonus** around a **point of interest** (default **x = 0**): encourages final stabilization where I want it.

> Intuition: additive rewards let “spinners” accumulate points unfairly; multiplicative shaping **amplifies** true stabilization quality and **penalizes** late/rough solutions.

## Escaping Local Maxima (Exploration Strategy)
Early runs often got stuck in **local optima**. I added:
- **Novelty-driven selection** + **many immigrants** in early generations to map the parameter space widely and keep diverse candidates alive.
- **Adaptive exploration kick**: if **no fitness improvement** for ~30 generations, **increase mutation σ** and the **immigrant fraction** temporarily to jump out of plateaus.

## Robustness via Multi-Start Simulation
To avoid overfitting to a single initial condition, **each generation evaluates individuals 3×** with slightly perturbed starts (Gaussian noise around the pendulum’s down position).  
- **Individual score = average of the 3 runs.**  
- This improved generalization and reduced brittle, over-specialized policies.

## Results (placeholders)
Add your media here once you export it from your simulation:
- **Upright recovery demo**  
  ![](/assets/images/animation.gif)
- **State values** 
  ![](/assets/images/figure_2.png)
- **Scoring function**
  ![](/assets/images/figure_3.png)

## Implementation Notes
- **Termination + shaping** order matters; apply shaping **only** while upright to avoid rewarding spins.
- Keep output `F` bounded and scale to your simulator’s units.
- Start with a small hidden layer; too big slows the GA and increases local minima density.
- Log **best-in-generation** scores and **moving best** to detect plateaus (trigger the exploration kick).

## What I Learned
- Multiplicative reward shaping can **dramatically** reduce “fake progress” from unstable behaviors.
- **Novelty + immigrants** is a simple, effective cure for early stagnation.
- Evaluating across **multiple initializations** is essential for controllers that **transfer** beyond a single setup—very relevant for TVC-like systems.

## Next Steps
- Try **CMA-ES** or **policy gradients** as a comparison baseline.
- Add **actuator limits** and friction to increase realism.
- Port controller to a **TVC gimbal sim** with similar state/action design.
