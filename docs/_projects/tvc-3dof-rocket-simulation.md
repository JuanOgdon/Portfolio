---
layout: page
title: "TVC Rocket Simulation — 3-DOF (planar) with Guidance & PID Control"
date: 2025-09-24
tags: [controls, guidance, simulink, tvc, aerodynamics]
status: in-progress
summary: "Planar 3-DOF rocket simulator in Simulink with thrust-vector control, aero forces/moments, actuator limits, waypoint guidance, and PID."
---

## TL;DR
- Goal: Build a realistic 3-DOF (x–z plus pitch) TVC rocket simulator to study attitude control and guidance.
- Plant: Simulink 3DOF (body axes) with thrust vectoring, drag, lift, aerodynamic moment, altitude-varying air density, and a small fixed nozzle misalignment.
- Data: Mass/inertia and CP–CG lever from OpenRocket; CL(alpha) and CD(alpha) curves fitted from RASAero II.
- Control: Waypoint guidance → error → PID (Ziegler–Nichols seed + manual tuning); gimbal angle/rate limits.
- Results: Two-waypoint run with final miss < 0.5 m at WP1 and < 1 m at WP2.

---

## Problem & Setup
This is a **planar** rocket model (x–z) with pitch dynamics. States and frames follow Simulink’s **3DOF (body axes)** block.

**Forces and moments**
- Thrust-vector components: Fx = T * sin(phi), Fz = T * cos(phi).
- Aerodynamics: drag along the free-stream direction and lift perpendicular to it.
- Aerodynamic moment from the CP–CG lever arm and a small fixed nozzle misalignment.
- Dynamic pressure q = 0.5 * rho * v^2 with air-density rho that decreases with altitude.
- Parameters (mass, inertia, CP/CG, reference area) from OpenRocket; CL(alpha) ~ linear near small alpha, CD(alpha) ~ quadratic.

**Actuators & limits**
- Gimbal angle: ±5°; gimbal rate: ±5°/s.
- Engine thrust approximated quasi-constant (G12-class), burn ≈ 8.5 s.

---

## Guidance & Control
- **Waypoint guidance:** compute the line-of-sight angle to the waypoint  
  theta_g = arctan((x_wp − x) / (z_wp − z)) and track a commanded attitude  
  theta_cmd = 90° − theta_g.
- **PID controller:** tuned via Ziegler–Nichols (no-overshoot variant) as a starting point, then refined manually to reduce steady-state error and settling time.
- **Abort logic:** stop the run if altitude z ≤ 0 (ground hit) or if absolute angle of attack |alpha| exceeds a critical limit.

---

## Cool ideas / mitigations
- Misalignment-torque model: include a fixed nozzle offset to capture build tolerances and create a realistic attitude bias the controller must reject.
- Altitude-varying density: prevents fake constant-velocity artifacts; drag effects (e.g., lower apogee) match expectations.
- Sequential waypoints: switch to WP2 only after WP1 is reached, with explicit error checks.
- Gimbal angle/rate saturation: honest controller tuning with authority limits at higher dynamic pressure.
- Limited rate of change on the reference derivative (D-term protection): reduces spikes when the active waypoint changes and helps avoid divergence.

---

## What I learned
- Modeling non-idealities (misalignment, actuator limits, abort conditions) early makes controllers more robust.
- Altitude-dependent aerodynamics are critical; otherwise performance metrics can be misleading.
- With waypoint-based navigation, discontinuities in the reference when switching waypoints can excite the D-term; limiting its rate of change improves stability.

---

## Images (Simulink and forces diagrams)
**Results after tuning PID**

<img width="1519" height="663" alt="Simulink overview" src="https://github.com/user-attachments/assets/2add10bd-0223-4214-b50e-fc29d7778d64" />

**Misalignment diagram**

<img width="915" height="855" alt="Misalignment diagram" src="https://github.com/user-attachments/assets/c1f0532b-3f58-4f2a-b6f3-a60500995fef" />
