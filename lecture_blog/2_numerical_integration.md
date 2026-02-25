
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

All integration methods are based on Taylor expansions. If we know position, velocity, acceleration, and higher derivatives at time t, we can approximate the state at t + Δt:

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
- Simple to implement

**Cons:**
- Not time-reversible: running the simulation backwards does not recover the original trajectory
- Not very accurate: energy drifts systematically over time

For production MD, Euler is not used.

## The Verlet algorithm

By adding the Taylor expansion forward and backward in time and cancelling the odd terms, the velocity drops out entirely:

```
Verlet algorithm:
r(t + Δt) = 2r(t) - r(t - Δt) + Δt²·a(t)
```

**Pros:**
- Straightforward implementation

**Cons:**
- Numerical precision: adding the small correction term Δt²·a(t) to the difference of two large terms 2r(t) - r(t - Δt) causes loss of numerical precision
- No explicit velocity term (velocities must be recovered separately, e.g. v(t) ≈ [r(t+Δt) - r(t-Δt)] / 2Δt)
- Not self-starting: requires r(t - Δt) at the start, which is not directly available

## The Leapfrog algorithm

Leapfrog avoids the precision problem by explicitly tracking velocities at half-integer time steps, so positions and velocities "leapfrog" over each other:

```
Leapfrog algorithm:
v(t + ½Δt) = v(t - ½Δt) + Δt·a(t)
r(t + Δt)  = r(t) + Δt·v(t + ½Δt)
```

**Pros:**
- Explicitly includes velocity term
- No calculation of difference of large numbers (avoids the Verlet precision problem)
- Exactly time-reversible
- Preserves energy well
- Straightforward implementation

**Cons:**
- Positions and velocities are not synchronized: they are evaluated at different times, so kinetic and potential energy cannot be computed at the same instant
- Not self-starting: requires v(t - ½Δt) at the start, which must be estimated

## The Velocity-Verlet algorithm

Velocity-Verlet reformulates the integration so that positions and velocities are synchronized at every step:

```
Velocity-Verlet algorithm:
r(t + Δt) = r(t) + Δt·v(t) + ½Δt²·a(t)
a(t + Δt) = f(r(t + Δt)) / m        ← recompute forces at new positions
v(t + Δt) = v(t) + ½Δt·[a(t) + a(t + Δt)]
```

In practice this is a three-stage procedure: compute new positions, recompute forces, then compute new velocities.

**Pros:**
- Synchronized positions and velocities: total energy can be computed at every step
- Self-starting: only requires r(t) and v(t) at the start
- Explicitly includes velocity term
- No calculation of difference of large numbers
- Straightforward implementation

**Cons:**
- Requires a bit more memory than Verlet or Leapfrog (must store both a(t) and a(t+Δt) simultaneously during the velocity update)

## Comparison

| Algorithm       | Time-reversible | Preserves energy | Synchronized r and v | Self-starting |
|-----------------|-----------------|------------------|----------------------|---------------|
| Euler           | No              | No (drifts)      | Yes                  | Yes           |
| Verlet          | Yes             | Yes              | No (no v term)       | No            |
| Leapfrog        | Yes             | Yes              | No                   | No            |
| Velocity-Verlet | Yes             | Yes              | Yes                  | Yes           |

Leapfrog and Velocity-Verlet are the standard choices. The lecture highlights both as suitable integrators.

## Periodic boundary conditions

To simulate bulk material, the simulation box is replicated infinitely in all directions. Each atom interacts with atoms in its own box and with image atoms in neighboring replicas. There is no surface: an atom that leaves through one face immediately re-enters through the opposite face.

In practice, PBC uses the minimum image convention: for each pair of atoms, we use the shortest possible distance, folding coordinates into the range [-L/2, L/2]:

```
Coordinates: if r_ix >= L/2, replace by r_ix - L
             if r_ix <= -L/2, replace by r_ix + L
Interactions: same rule applied to r_ijx
```

**Important limitation:** PBC limits the interaction range to L/2. Long-range interactions (e.g. Coulomb) need special treatment and cannot simply use the cutoff approach.

Advantages of PBC:
- Eliminates surface artifacts
- Enables simulation of bulk properties with a small number of atoms

Disadvantages:
- Artificial periodicity: if the box is too small, an atom interacts with its own image
- The box size must be larger than twice the interaction cutoff
- Long-range interactions require special methods (e.g. Ewald summation)

## Key answers

**Q4. Name two numerical integration algorithms and their advantages/disadvantages.**

Leapfrog: explicitly includes velocity term, no subtraction of large numbers, exactly time-reversible, preserves energy well, straightforward implementation. Drawback: positions and velocities are not synchronized (kinetic and potential energy cannot be computed at the same time), and it is not self-starting.<br>
Velocity-Verlet: synchronized positions and velocities (total energy checkable at every step), self-starting, explicitly includes velocity term, straightforward implementation. Drawback: requires slightly more memory than Verlet or Leapfrog.

**Q5. What are periodic boundary conditions and why are they used in MD quite often?**

The simulation box is replicated infinitely in all directions so there is no surface. Atoms leaving one face re-enter from the opposite face. Used because a realistically-sized box would be almost entirely surface, making bulk properties impossible to measure. Important caveat: PBC limits the interaction range to L/2, so long-range interactions need special treatment.
