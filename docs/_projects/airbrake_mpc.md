# Rocket Airbrake: MPC Controller + Kalman Filter

**Status:** In development — competition rocket project

This project is developing a deployable airbrake system for a competition rocket, combining physical characterization of the aerodynamic device with a Model Predictive Control (MPC) apogee-targeting controller and a Kalman filter for state estimation.

---

## The problem

In competition rocketry, hitting a target apogee altitude precisely — rather than just flying as high as possible — is a core challenge. The rocket's propulsion is fixed by the motor, so the only in-flight control authority is drag modulation. An airbrake (a set of deployable fins or flaps) increases aerodynamic drag when extended, allowing the controller to bleed off excess kinetic energy and hit the target altitude.

This is a closed-loop estimation and control problem: the controller needs to know the rocket's current state (altitude, velocity) accurately, predict how the trajectory will evolve under different airbrake deployments, and choose the deployment that hits the target.

---

## System components

### Physical modeling and characterization

Before any controller can be designed, the aerodynamic effect of the airbrake must be quantified. This involves modeling how the drag coefficient changes as a function of airbrake extension angle and flight conditions, and characterizing the actuator dynamics (how quickly the airbrake can deploy and retract).

### Kalman filter — state estimation

Sensors on board (accelerometers, barometric altimeter) are noisy and subject to bias. A Kalman filter fuses these measurements with a dynamic model of the rocket to produce a clean, continuous estimate of altitude and velocity throughout the flight. This estimated state is what the controller acts on.

### MPC controller — apogee targeting

Model Predictive Control works by: at each timestep, using a model of the rocket's dynamics to predict the trajectory over a future horizon for different control inputs (airbrake positions), then selecting the input sequence that brings the rocket closest to the target apogee.

The key advantage of MPC here is that it handles the nonlinear relationship between airbrake position, drag, and trajectory naturally — it just simulates forward and picks the best option. It also respects the physical constraints on the airbrake (can't extend beyond max angle, limited actuation speed) as part of the optimization.

---

## Current status

- Physical modeling and aerodynamic characterization: in progress
- Simulation of full flight (motor burn + coast + apogee): built
- Kalman filter: designed
- MPC controller: designed
- Integration and hardware-in-the-loop testing: upcoming

---

## What makes this interesting

This project sits at the intersection of several things I care about: real hardware constraints, nonlinear dynamics, and algorithms that have to work reliably in a short time window (the coast phase between motor burnout and apogee is typically a few seconds). It's also directly connected to the TVC simulation work — the same tools and modeling approaches, applied to a different control problem on a real physical vehicle.

---

**Tools used:** MATLAB/Simulink, Python, physical test setup
