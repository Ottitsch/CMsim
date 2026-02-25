
# What you can measure from an MD trajectory

An MD trajectory is a time series of all atom positions and velocities. From it, you can compute a surprisingly wide range of physical and chemical properties, both structural and dynamical.

## Exam questions

- **Q10.** Name a few static and dynamic properties that can be obtained from an MD trajectory.
- **Q11.** Why can we compute dynamic (=non-equilibrium) properties from equilibrium trajectories?
- **Q12.** What are possible sources of error in analyzing an MD trajectory? How can we avoid them?

## Static vs dynamic properties

**Static (structural) properties** depend only on instantaneous configurations. They do not require time ordering and can be computed as time averages over independent snapshots:

- Total energy (kinetic + potential)
- Temperature and pressure
- Radial distribution function g(r)
- Structure factor S(q) (Fourier transform of g(r), measurable by X-ray/neutron diffraction)
- Coordination numbers and local structure
- Density and volume

**Dynamic (transport) properties** describe how the system evolves over time. They require time-correlated data, i.e., pairs of snapshots separated by a time lag:

- Self-diffusion coefficient D (how fast atoms wander through the liquid)
- Viscosity η (resistance to flow)
- Thermal conductivity λ (heat transport)
- Electrical conductivity σ (charge transport, for ionic systems)

The distinction matters for how you compute them: static properties just need averages, while dynamic properties need autocorrelation functions.

## Why equilibrium simulations give dynamic properties

It might seem paradoxical that you can compute non-equilibrium transport properties (how the system responds to a gradient) from an equilibrium trajectory (where there is no gradient). The connection comes from linear response theory, specifically the Green-Kubo formalism.

The key idea is the fluctuation-dissipation theorem: the same microscopic fluctuations that drive equilibrium fluctuations also govern the system's response to a small perturbation. In other words, the way spontaneous fluctuations decay at equilibrium is mathematically identical to the way the system relaxes back after a small perturbation.

This means: if you watch how velocity fluctuations in an equilibrium simulation decay over time, you get exactly the same information as if you applied a small force and measured the resulting drift.

Formally, Green-Kubo relations express transport coefficients as integrals of equilibrium autocorrelation functions:

```
D = (1/3) ∫₀^∞ <v(0) · v(t)> dt      (diffusion from velocity autocorrelation)

η = (V / k_B T) ∫₀^∞ <P_xy(0) · P_xy(t)> dt   (viscosity from stress autocorrelation)
```

where <v(0)·v(t)> is the velocity autocorrelation function (VACF) and P_xy is an off-diagonal pressure tensor element.

## Computing diffusion: two equivalent approaches

The diffusion coefficient describes how quickly atoms move away from their starting position. Two routes give the same result:

**Einstein relation** (via mean squared displacement):

```
D = lim_{t→∞} <|r(t) - r(0)|²> / (6t)
```

Plot the mean squared displacement MSD as a function of time. At long times it grows linearly. The slope divided by 6 (for 3D) is D.

**Green-Kubo relation** (via velocity autocorrelation):

```
D = (1/3) ∫₀^∞ <v(0) · v(t)> dt
```

Compute the VACF: for each atom, multiply its velocity at time 0 by its velocity at time t, average over all atoms and all time origins, and integrate.

The VACF starts at <v²> (positive) and decays to zero. For a liquid, it typically shows a small negative dip (the "cage effect" (atoms bounce back from their neighbors)) before going to zero.

## Error sources in MD trajectory analysis

Three main sources of error appear in MD analysis:

**1. Insufficient sampling (short simulation)**
If the simulation is too short, the time average does not converge to the ensemble average. Rare events and slow processes are not captured. The measured property fluctuates and has high statistical uncertainty. Solution: run longer simulations.

**2. Correlated data**
Consecutive MD frames are not independent. Atom positions change only slightly between adjacent time steps. If you estimate the error using the standard error of the mean over all frames, you will severely underestimate the true error because you are not counting independent samples. Solution: use block averaging (divide the trajectory into blocks, compute the property per block, estimate error from the spread across blocks) or compute the autocorrelation time and use it to determine the number of statistically independent samples.

**3. Non-equilibrated initial state**
If you start measurement too early, the system is still relaxing from its initial configuration. Properties from that period reflect the initial state, not the equilibrium state of interest. Solution: always equilibrate first, monitor total energy or temperature convergence, and only start production measurement after equilibration is confirmed.

## The autocorrelation function

Many dynamic observables involve the autocorrelation function C(t) of some quantity A:

```
C(t) = <A(0) · A(t)>
```

C(0) = <A²> (the variance). C(t) decays to zero at large t for an ergodic system.

The autocorrelation time τ_A gives a rough measure of memory: for t >> τ_A the value at time t is uncorrelated with the value at time 0. This is the timescale on which you need to separate samples to treat them as independent.

The practical consequence: if your simulation is only a few autocorrelation times long, your error estimate will be dominated by statistical noise and will not be reliable.

## Key answers

**Q10. Name a few static and dynamic properties that can be obtained from an MD trajectory.**

Static: energy, temperature, pressure, radial distribution function g(r), structure factor, coordination numbers.<br>
Dynamic: self-diffusion coefficient, viscosity, thermal conductivity, electrical conductivity.

**Q11. Why can we compute dynamic (non-equilibrium) properties from equilibrium trajectories?**

The fluctuation-dissipation theorem states that equilibrium fluctuations contain the same information as the system's response to a small perturbation. Green-Kubo relations exploit this: transport coefficients equal integrals of equilibrium time-autocorrelation functions (e.g. diffusion from the velocity autocorrelation function).

**Q12. What are possible sources of error in analyzing an MD trajectory? How can we avoid them?**

(1) Short simulation: the time average has not converged. Fix by running longer.<br>
(2) Correlated data: consecutive frames are not independent, so naive error estimates are too small. Fix by block averaging or computing the autocorrelation time.<br>
(3) Non-equilibrated start: early frames reflect the initial configuration, not the target state. Fix by equilibrating first and discarding that period before analysis.
