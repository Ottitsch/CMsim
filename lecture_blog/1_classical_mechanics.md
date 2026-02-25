
# The two things every MD simulation needs

Molecular dynamics is, at its core, a simulation of atoms moving over time. To run it, you need exactly two ingredients: a way to compute forces from positions, and a way to propagate positions forward in time.

## Exam questions

- **Q1.** What are the two fundamental functions we need to know to run MD?
- **Q2.** Why is Newtonian mechanics relevant for MD?
- **Q3.** Which observables are conserved in a closed system? Why?

## The two fundamental functions

For an MD simulation, we need to know:

1. How atoms interact: given coordinates r, obtain forces f(r)
2. How atoms move: given forces f, obtain coordinates at the next time step r(t + Δt)

The force function describes the potential energy surface of the system. Once you know where all atoms are, you can compute how strongly and in which direction each atom is being pushed. The integrator then uses those forces to advance all positions by one small time step Δt and repeat.

Every MD simulation, regardless of how complex, reduces to this loop: compute forces, update positions, compute forces, update positions, thousands or millions of times.

## Why Newtonian mechanics

Quantum mechanics is the more fundamental theory, but it is computationally intractable for systems of thousands of atoms. Classical mechanics is the asymptote of quantum mechanics for increasingly large systems. For atoms at room temperature that are not too light and not involved in bond breaking or electronic excitations, the classical approximation is good enough.

Newtonian mechanics is the earliest and simplest classical formulation. It is also the one that maps most directly onto a computer simulation: given forces and masses, you get accelerations via Newton's second law, which you can integrate numerically.

The equation of motion for atom i with mass m_i is:

```
f_i(t) = m_i * a_i(t)
```

This is a second-order ordinary differential equation. Given positions and velocities at time t, numerical integration gives you positions and velocities at t + Δt.

## From one particle to many

A single point particle is fully described by position x(t), velocity v(t) = dx/dt, acceleration a(t) = dv/dt, and force f(t) = m*a.

For N particles, the state of the system is the full position vector x(t) = (x_1, y_1, z_1, ..., x_N, y_N, z_N) and likewise for velocities. The total force on particle i is the vector sum of pairwise interactions with all other particles j ≠ i.

## What is conserved in a closed system

In a closed system with no external forces, three quantities are conserved. The connection between conservation laws and symmetries comes from Noether's theorem:

**Total mechanical energy E = E_k + E_pot** is conserved because the laws of physics are symmetric under translations in time (time translational symmetry). The system evolves the same way regardless of when you start the clock.

**Linear momentum P** is conserved because the laws of physics are symmetric under translations in space (translational symmetry). There is no preferred origin.

**Angular momentum L** is conserved because the laws of physics are symmetric under rotations in space (rotational symmetry). There is no preferred direction.

In practice, these conservation laws serve as diagnostic tools. If total energy drifts during an MD simulation, the integration is not accurate enough. If momentum grows, you may have a bug in your force calculation.

## The force is the negative gradient of the potential

For a conservative force field, the force on each atom is the negative gradient of the potential energy:

```
f = -∇E_pot
f_i = -∂E_pot / ∂x_i
```

This means once you have a potential energy function, you get forces for free by differentiation. The choice of potential energy function is what distinguishes different types of MD simulations.

## Key answers

**Q1 — Two fundamental functions:**
(1) Force function: given positions r, compute forces f(r). (2) Integrator: given forces f, compute positions r(t+Δt).

**Q2 — Why Newtonian mechanics:**
Newtonian mechanics is the classical asymptote of quantum mechanics for large, heavy systems — it is a good approximation for atoms at room temperature and is simple enough to integrate numerically for thousands of atoms.

**Q3 — Conserved observables in a closed system:**
Total mechanical energy (time-translational symmetry), linear momentum (translational symmetry), and angular momentum (rotational symmetry) — each conservation law follows from a symmetry of the equations of motion via Noether's theorem.
