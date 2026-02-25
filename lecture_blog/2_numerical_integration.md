
# Moving atoms forward in time

Newton's equations of motion are a system of second-order differential equations. For more than two bodies there is no analytic solution, so we integrate them numerically using finite time steps. The choice of algorithm matters: a poor integrator wastes energy and produces physically wrong trajectories.

## Exam questions

- **Q4.** Name two numerical integration algorithms and their advantages/disadvantages.
- **Q5.** What are periodic boundary conditions and why are they used in MD quite often?

## The integration loop

MD trajectories are built step by step:

1. Compute the force on each particle as the vector sum of its interactions with all other particles
2. Obtain the acceleration from Newton's second law: a = f/m
3. Combine acceleration, positions, and velocities at time t to get positions and velocities at t + Δt
4. Repeat

All integration methods are based on Taylor expansions. If we know position, velocity, acceleration, jerk, and higher derivatives at time t, we can approximate the state at t + Δt:

```
r(t + Δt) = r(t) + Δt·v(t) + ½Δt²·a(t) + ⅙Δt³·j(t) + ...
v(t + Δt) = v(t) + Δt·a(t) + ½Δt²·j(t) + ...
```

The question is where to truncate and how to arrange the computation.

## The Euler algorithm

The simplest approach is to truncate the Taylor expansion after the acceleration term:

```
Euler algorithm:
r(t + Δt) = r(t) + Δt·v(t) + ½Δt²·a(t)
v(t + Δt) = v(t) + Δt·a(t)
```

**Pros:**
- Simple to implement and understand

**Cons:**
- Not time-reversible: running the simulation backwards does not recover the original trajectory
- Not very accurate: energy drifts systematically over time because the truncation error accumulates in one direction
- Not symplectic: does not preserve the geometric structure of phase space

For production MD, Euler is not used.

## The Velocity-Verlet algorithm

Velocity-Verlet is the standard algorithm in modern MD codes. It updates positions and velocities in a symmetric way:

```
Velocity-Verlet algorithm:
r(t + Δt) = r(t) + Δt·v(t) + ½Δt²·a(t)
a(t + Δt) = f(r(t + Δt)) / m        ← recompute forces at new positions
v(t + Δt) = v(t) + ½Δt·[a(t) + a(t + Δt)]
```

**Pros:**
- Time-reversible: running the simulation backwards recovers the original trajectory
- Symplectic: conserves a modified energy, so total energy fluctuates around a constant but does not drift
- Good energy conservation even for moderately large time steps
- Only one force evaluation per step (same computational cost as Euler)
- Positions and velocities are available at the same time step, making energy computation straightforward

**Cons:**
- Slightly more complex to implement than Euler or Leapfrog
- Requires storing both the old and new accelerations simultaneously during the velocity update

The symplectic property is the key reason Velocity-Verlet is preferred. A symplectic integrator preserves the geometric structure of phase space and keeps long simulations stable.

## The Leapfrog algorithm

Leapfrog is mathematically equivalent to Velocity-Verlet but stores velocities at half-integer time steps:

```
Leapfrog algorithm:
v(t + ½Δt) = v(t - ½Δt) + Δt·a(t)
r(t + Δt)  = r(t) + Δt·v(t + ½Δt)
```

**Pros:**
- Time-reversible and symplectic: same accuracy and stability as Velocity-Verlet
- Slightly simpler to implement than Velocity-Verlet
- Memory efficient: only one set of velocities needs to be stored at a time

**Cons:**
- Positions and velocities are offset by half a time step and are never known at the same instant
- Computing instantaneous kinetic energy requires averaging v(t - ½Δt) and v(t + ½Δt), adding a small complication
- Less intuitive conceptually than Velocity-Verlet

In practice, Leapfrog and Velocity-Verlet produce identical trajectories. Most MD codes use one or the other based on convention.

## Comparison: Euler vs Velocity-Verlet vs Leapfrog

| Algorithm       | Time-reversible | Energy conserving | Cost |
|-----------------|-----------------|-------------------|------|
| Euler           | No              | No (drifts)       | Low  |
| Velocity-Verlet | Yes             | Yes (symplectic)  | Low  |
| Leapfrog        | Yes             | Yes (symplectic)  | Low  |

All three have the same computational cost per step. The difference is entirely in accuracy and stability, not speed.

## Periodic boundary conditions

A simulation box of 1000 atoms has a radius of roughly 3-4 nm. That is almost entirely surface. Real materials are bulk, so surface effects would completely dominate the simulation and give wrong results.

Periodic boundary conditions (PBC) solve this by replicating the simulation box infinitely in all directions. Each atom interacts with atoms in its own box and with image atoms in neighboring replicas. Effectively, there is no surface: an atom that leaves through one face immediately re-enters through the opposite face.

In practice, PBC uses the minimum image convention: for each pair of atoms, we use the shortest possible distance between them, considering all periodic images. For a box of length L, this means distances are folded into the range [-L/2, L/2].

```
Minimum image:
if |Δx| > L/2: Δx = Δx - sign(Δx) * L
```

Advantages of PBC:
- Eliminates surface artifacts
- Enables simulation of bulk properties with a small number of atoms
- Required for computing properties like diffusion or viscosity that require translation invariance

Disadvantages:
- Artificial periodicity: if the box is too small, an atom interacts with its own image
- The box size must be larger than twice the interaction cutoff
- Periodic systems cannot model truly isolated molecules or interfaces without care

## Key answers

**Q4 — Two integration algorithms:**
Euler: simple to implement, but not time-reversible and energy drifts — not used in practice. Velocity-Verlet: time-reversible, symplectic (energy stable), same cost as Euler — the standard choice. Leapfrog: equivalent to Velocity-Verlet, equally accurate, but positions and velocities are offset by half a time step.

**Q5 — Periodic boundary conditions:**
The simulation box is replicated infinitely in all directions so there is no surface. Atoms leaving one face re-enter from the opposite face. Used because a realistically-sized box would be almost entirely surface, making bulk properties impossible to measure.
