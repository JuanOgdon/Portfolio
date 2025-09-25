---
layout: page
title: "TVC Rocket Simulation — 3-DOF (planar) with Guidance & PID Control"
date: 2025-09-24
tags: [controls, guidance, simulink, tvc, aerodynamics, openrocket, rasaero]
status: finished
summary: "Planar 3-DOF rocket simulator in Simulink with thrust-vector control, aero forces/moments, actuator limits, waypoint guidance, and PID."
---

## TL;DR
- **Goal:** Build a realistic 3-DOF (x–z translation + pitch) TVC rocket sim to study attitude control and guidance.  
- **Plant:** Simulink 3DOF (body axes), with **thrust vectoring**, **drag**, **lift**, **aero moment**, density varying with altitude, and **nozzle misalignment**.  
- **Data:** Mass/inertia/CP-CG lever from **OpenRocket**; \(C_L(\alpha), C_D(\alpha)\) from **RASAero II** (fit to functions).  
- **Control:** Waypoint guidance → error → **PID** (Ziegler-Nichols seed + manual refinement); **gimbal angle/rate** limits.  
- **Results:** Correct drag behavior (lower apogee), passive aero stability (α→0), two-waypoint run with final errors ≈ **0.38 m** and **2.28 m** at WP1/WP2, respectively. :contentReference[oaicite:0]{index=0}

---

## Problem & Setup
This is a **planar** rocket model (x–z) with pitch dynamics. States and frames follow Simulink’s **3DOF (body axes)** block. Forces: weight, **TVC thrust components** \(F_x= T\sin\varphi,\, F_z= T\cos\varphi\); **aero**: \(D\) along free-stream, \(L\) perpendicular; **moments** from CP–CG lever and **nozzle misalignment** (small fixed offset). Aero uses dynamic pressure \(q=\tfrac12\rho v^2\) with **\(\rho(h)\)** so terminal effects aren’t constant-V artifacts. Parameters (mass, inertia, CP/CG, reference area) come from OpenRocket; \(C_D(\alpha)\) ~ quadratic, \(C_L(\alpha)\) ~ linear from RASAero II fits. :contentReference[oaicite:1]{index=1}

**Actuators & limits**
- Gimbal angle: ±5°; rate: ±5°/s.  
- Engine thrust profile approximated quasi-constant (G12-class), burn ≈ 8.5 s. :contentReference[oaicite:2]{index=2}

---

## Guidance & Control
- **Waypoint Guidance:** compute desired line-of-sight angle \(\theta_g = \arctan\frac{x_\text{wp}-x}{z_\text{wp}-z}\); track **\(\theta_\text{cmd} = \tfrac{\pi}{2}-\theta_g\)**.  
- **PID:** tuned via **Ziegler-Nichols (no-overshoot)** to get initial \(K_p, K_i, K_d\), then hand-refined to reduce steady-state error and settling time. :contentReference[oaicite:3]{index=3}
- **Abort logic:** terminate if \(z \le 0\) (ground) or \(|\alpha| > \alpha_\text{crit}\) to avoid invalid aero regime and unsafe attitudes. :contentReference[oaicite:4]{index=4}

---

## Cool ideas / mitigations
- **Misalignment torque model:** added fixed nozzle offset to capture build tolerances → realistic attitude bias & recovery requirement. :contentReference[oaicite:5]{index=5}
- **Altitude-varying density:** prevents fake constant terminal-velocity behavior; drag effects match expectations (e.g., lower apogee with drag). :contentReference[oaicite:6]{index=6}
- **Sequential waypoints:** reaches far targets by updating to WP2 after WP1 with explicit error checks. :contentReference[oaicite:7]{index=7}
- **Gimbal angle/rate saturation:** makes controller tuning honest; shows authority limits at higher dynamic pressure. :contentReference[oaicite:8]{index=8}
- **Derivative Term in reference** limited with a rate of change to make it more stable and avoid divergence/instability when changing guidance waypoint.

---

## What I learned
- Modeling **non-idealities** (misalignment, actuator limits, α-abort) early makes controllers more robust.  
- Altitude-dependent aero is critical; otherwise performance metrics are misleading.
- With a waypoint based navigation, the discontinuities in the reference point once the waypoint changes can make the k_d term have a spike a start divergence in the stability.

---

## Images of the Simulink and Forces diagrams

Results after tuning PID

<img width="1519" height="663" alt="image" src="https://github.com/user-attachments/assets/2add10bd-0223-4214-b50e-fc29d7778d64" />

Missalignment diagram

<img width="915" height="855" alt="image" src="https://github.com/user-attachments/assets/c1f0532b-3f58-4f2a-b6f3-a60500995fef" />


