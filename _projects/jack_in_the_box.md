---
layout: project
title: Jack-in-the-Box Dynamics
tags: Dynamics, Simulation
gif: /assets/gifs/jack_demo.gif
---

<iframe class="video" src="https://www.youtube.com/embed/isUHyNV9Cu8" allowfullscreen></iframe>

---

## Overview
This project investigates the **nonlinear dynamics of a Jack-in-the-Box mechanism**, focusing on contact, spring, and impact interactions.  
The system consists of a mass–spring–hinge assembly that undergoes large-amplitude motion when released, exhibiting **hybrid dynamics**—continuous spring oscillation combined with discrete impact events.  

The objective was to build a physics-accurate model and simulate the box-lid motion and internal “pop-up” dynamics to better understand **impact timing, damping effects, and energy transfer** between components.

---

## Mathematical Modeling
1. **System Representation**
   - Modeled as a **planar multi-body system** with the lid, base, and spring mass linked through revolute and prismatic joints.
   - Used **Lagrangian mechanics** to derive equations of motion:
     \[
     \frac{d}{dt}\left(\frac{\partial L}{\partial \dot{q}}\right) - \frac{\partial L}{\partial q} = Q
     \]
     where \( L = T - V \) and \( Q \) represents non-conservative generalized forces (friction, damping, contact).

2. **Contact and Impact Handling**
   - Implemented **event-based switching** to detect lid-mass contact and apply impulse-momentum conservation.
   - Energy loss modeled via a **coefficient of restitution** and damping ratio calibration.

3. **Simulation Framework**
   - Implemented in **Python (SymPy + SciPy integrators)** for symbolic derivation and numerical integration.
   - Real-time visualization and playback through **Matplotlib animation** and **Open3D** for geometric rendering.

---

## Technical Highlights
- **Frameworks:** Python, SymPy, SciPy, Open3D, Matplotlib  
- **Physics Engine:** Custom Lagrange-based solver with impact detection  
- **Key Outputs:** Position, velocity, and energy plots; phase-space trajectories  
- **Runtime:** ~60 fps real-time simulation on a single CPU core  
- **Outcome:** Verified energy consistency and realistic rebound patterns across multiple damping and stiffness parameters.

---

## Key Learnings
- Developed a deeper understanding of **hybrid dynamical systems** combining continuous and discrete behaviors.  
- Implemented an efficient **event-triggered ODE solver** with symbolic-to-numeric translation using SymPy.  
- Visualized motion profiles and validated stability under different spring constants and damping ratios.  
- Learned how to extend classical dynamics methods to systems involving **contact and constraint switching**.

---

## Future Work
- Extend model to **3D multi-link dynamics** and include **compliant contact surfaces**.  
- Port solver to **C++ or Rust** for faster real-time integration.  
- Integrate with **Isaac Sim or Mujoco** for robotics-grade physics validation.  
- Explore **control laws** for stabilizing post-impact oscillations or energy recovery.

---

## Media
**Jack-in-the-Box Dynamics Demo:**  
<iframe class="video" src="https://www.youtube.com/embed/isUHyNV9Cu8" allowfullscreen></iframe>  
> Simulates the coupled spring–lid system showing impact events, oscillatory motion, and energy dissipation.
