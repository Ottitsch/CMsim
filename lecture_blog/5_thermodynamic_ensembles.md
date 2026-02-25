
# Controlling temperature and pressure in MD

An MD simulation in isolation samples the microcanonical ensemble. But most real experiments happen at constant temperature and pressure, not constant energy. To simulate those conditions we need thermostats and barostats — algorithms that couple the system to a heat bath or pressure bath. The choice of algorithm affects not just the temperature but also whether the resulting trajectory can be used to compute dynamic properties.

## Exam questions

- **Q13.** Describe the microcanonical (NVE), canonical (NVT), and isobaric-isothermal (NPT) ensemble.
- **Q14.** How is temperature controlled in an MD run? Name at least two options and their advantages/disadvantages.
- **Q15.** For computing dynamic properties from an MD run, name one good and bad choice of a thermostat, and explain your choice.
- **Q16.** How is pressure controlled in an MD run? Name at least two options and their advantages/disadvantages.

## The three ensembles

Different macroscopic constraints define different statistical ensembles. Three are most important for MD:

**Microcanonical ensemble (NVE):** Number of particles N, volume V, and total energy E are all fixed. The system is perfectly isolated. This is what a plain MD simulation without any thermostat or barostat produces.

**Canonical ensemble (NVT):** N and V are fixed, but energy can fluctuate. Instead, the temperature T is specified. The system is in weak thermal contact with a heat bath. Allowing energy to distribute freely across microstates at a given T produces a Boltzmann distribution over energy states.

**Isobaric-isothermal ensemble (NPT):** N is fixed, but both energy and volume can fluctuate. Temperature T and pressure P are specified. The system is in contact with both a heat bath and a pressure bath. This is the ensemble that most closely matches typical lab conditions.

The hierarchy is: NVE → allow E to fluctuate → NVT → allow V to fluctuate → NPT.

Ensemble results can be transformed between ensembles only in the thermodynamic limit (infinite system size). For finite simulation boxes it is therefore better to directly sample the ensemble of interest.

## What constant temperature actually means

Temperature is defined at the molecular level by the average kinetic energy of all particles:

```
(3/2) N k_B T = E_k = (1/2) Σ_i v_i²
```

A key subtlety: **constant temperature does not mean the kinetic energy per particle is constant**. In a canonical ensemble the instantaneous temperature fluctuates. The velocities follow the Maxwell-Boltzmann distribution, which broadens with increasing temperature and narrows with increasing mass. The magnitude of these fluctuations is related to the heat capacity: a higher heat capacity means smaller temperature fluctuations for the same energy exchange.

Any thermostat that pins the instantaneous temperature to exactly T₀ at every step is therefore not producing a canonical ensemble — it is producing something physically wrong.

## Thermostats

### Velocity-rescaling (1971)

The simplest approach: at every time step, multiply all velocities by λ = √(T₀/T(t)) to force the temperature to exactly T₀.

**Pros:** Simple, easy to understand, straightforward to implement.

**Cons:**
- Introduces discontinuities in the momentum part of the phase space trajectory
- T(t) cannot fluctuate — it is clamped to T₀ at every step
- Does not generate rigorous canonical averages
- Artificially prolongs temperature differences among subsystems (only the average temperature is controlled)

### Berendsen thermostat (1984)

Instead of an instantaneous rescaling, couple the system to a heat bath so the rate of temperature change is proportional to the temperature difference:

```
dT(t)/dt = (1/τ)(T₀ − T(t))
```

with coupling parameter τ. For one time step this gives ΔT = (Δt/τ)(T₀ − T(t)), and the corresponding velocity scaling factor is:

```
λ = √(1 + (Δt/τ)(T₀/T(t) − 1))
```

When τ ≈ Δt, Berendsen reduces to velocity-rescaling. A typical choice is τ ≈ 0.4 ps for a 1 fs time step (τ ≈ 400 Δt).

**Pros:** Simple, easy to understand, straightforward to implement. Much smaller discontinuities than velocity-rescaling. T(t) can fluctuate depending on τ.

**Cons:**
- Does not generate rigorous canonical averages
- Artificially prolongs temperature differences among subsystems

### Stochastic rescaling (2007)

An extension of velocity-rescaling that fixes the canonical-averages problem. The idea: instead of forcing E_k(t) to exactly the target value E_{k,0}, draw the target stochastically from the canonical equilibrium distribution of kinetic energy:

```
P(E_{k,t}) ∝ E_{k,t}^{f/2 − 1} · exp(−E_{k,t} / k_B T)
```

where f is the number of degrees of freedom. The same principle can be applied to the Berendsen thermostat, giving a smoother trajectory.

**Pros:** Simple, easy to understand, straightforward to implement. Generates correct canonical averages. T(t) can fluctuate.

**Cons:** Stochastic — trajectories cannot be reproduced exactly.

### Andersen thermostat (1980)

A stochastic collision method. At random intervals, a particle is selected and its velocity is reassigned from the Maxwell-Boltzmann distribution. Between collisions the system runs at constant energy (NVE). The result is a Markov chain of short NVE segments.

The collision rate ν controls coupling strength. The probability that the next collision occurs between t and t + dt is P(t, ν) dt = ν e^{−νt} dt. Per time step the collision probability is ν Δt. Too low ν: under-samples the canonical distribution. Too high ν: temperature fluctuates too little.

**Pros:** Generates correct canonical averages.

**Cons:** Dynamic properties cannot be computed correctly because the random velocity reassignments destroy the physical velocity correlations. The dynamics is not physical.

### Nosé-Hoover thermostat (1985)

Make the heat bath a real part of the extended system by introducing an extra degree of freedom s with its own artificial coordinate and velocity. The extended system (with coordinates r_i, momenta p_i, time t) maps to the physical system (r'_i, p'_i, time t') via:

```
r'_i = r_i,    p'_i = p_i / s,    t' = ∫₀ᵗ (1/s) dt
```

The total energy of the extended system gains two new terms: a potential energy of the reservoir (f+1) k_B T₀ ln s (where f is the degrees of freedom of the physical system) and a kinetic energy of the reservoir (Q/2)(ds/dt)² with Q being the fictitious mass of the extra degree of freedom. Q controls coupling strength.

Simulating the microcanonical ensemble in the extended system produces the canonical ensemble in the physical system. The real velocity is v'_i = s · dr_i/dt, so s acts as a time scaling: dt' = dt/s.

**Pros:** Generates correct canonical averages. One of the most accurate and efficient methods for constant-temperature MD.

**Cons:** Difficult to understand and more difficult to implement. s can vary, so regular time intervals in the extended system map to uneven intervals in real time (solvable). Requires no external force and a fixed center of mass — otherwise can be non-ergodic or oscillatory. Remedy: Nosé-Hoover chains (a chain of thermostats with different masses).

### Langevin dynamics

Adds friction and a random force directly to the equations of motion, simulating the effect of an implicit solvent:

```
ma = F − mγv + R
```

The friction term mγv removes energy (like drag through a solvent). The random term R adds energy (like kicks from solvent molecules). The friction coefficient γ determines coupling strength.

**Pros:** Allows larger time steps.

**Cons:** Not deterministic. Changes the dynamics — usually slows it down. Dynamic properties computed from a Langevin trajectory reflect the dynamics of a particle in a solvent, not the intrinsic dynamics of the system.

### Thermostat comparison

| Thermostat | Type | Dynamics | Deterministic | T fluctuates | True NVT |
|---|---|---|---|---|---|
| Velocity-rescaling | v-scaling | sometimes correct | yes | no | no |
| Berendsen | v-scaling | sometimes correct | yes | yes | no |
| Stochastic rescaling | v-scaling | correct | no | yes | yes |
| Nosé-Hoover | v-scaling | correct | yes | yes | yes |
| Andersen | v-randomizing | wrong | no | yes | yes |
| Langevin | v-randomizing | wrong | no | yes | yes |

## Good and bad thermostats for dynamic properties

For computing dynamic properties (diffusion, viscosity, etc.) the thermostat must not interfere with the physical velocity correlations.

**Good choice: Nosé-Hoover thermostat.** It is deterministic and produces correct equations of motion for the extended system. The velocity correlations in the physical system are not disrupted by random kicks. The resulting dynamics are physical, and Green-Kubo integrals over velocity autocorrelation functions give correct transport coefficients.

**Bad choice: Andersen thermostat.** It randomly reassigns velocities at each collision event. This artificially decorrelates the velocities, destroying the very velocity autocorrelation functions that dynamic properties are computed from. For structural (static) properties Andersen is fine; for dynamic properties it is fundamentally broken.

The slide shows a concrete example: computing diffusion coefficients of water from MSD gives 4.26×10⁻⁹ m²/s with Langevin and 5.21×10⁻⁹ m²/s with Nosé-Hoover. The two thermostats give different values because Langevin modifies the dynamics by adding artificial friction, while Nosé-Hoover leaves the physical dynamics intact.

## Pressure control: barostats

Constant pressure is maintained by allowing the simulation box volume to fluctuate. A macroscopic system maintains constant pressure by changing its volume; the simulation does the same.

The magnitude of volume fluctuations is related to the isothermal compressibility κ:

```
κ = (1 / k_B T) · (⟨V²⟩ − ⟨V⟩²) / ⟨V²⟩
```

An easily compressible substance (large κ) undergoes large volume fluctuations. This matters for box size: for an ideal gas (κ ≈ 1 atm⁻¹), a 2 nm cube at 300 K fluctuates by 18.1 nm³ in volume — a cell-length fluctuation of 2.6 nm — which is larger than the box itself. An ideal gas simulation box of 2 nm is therefore not a reasonable simulation. For water (κ ≈ 44.75×10⁻⁶ atm⁻¹) the same box fluctuates by only 0.121 nm³ (0.49 nm cell length), which is manageable.

### Volume-scaling

The simplest approach: directly rescale the box volume (and all coordinates with it) to the target pressure. Not exactly NPT because it does not reproduce the correct volume fluctuation distribution.

### Berendsen barostat (1984)

Analogous to the Berendsen thermostat. Couple the system to a pressure bath so the rate of pressure change is proportional to the difference from the target:

```
dP(t)/dt = (1/τ_P)(P₀ − P(t))
```

with coupling parameter τ_P. This determines a scaling factor λ that rescales the volume and thus all atomic coordinates as r'_i = λ^{1/3} r_i.

**Pros:** Simple, easy to understand, straightforward to implement.

**Cons:** Does not generate rigorous NPT averages (same problem as Berendsen thermostat).

### Stochastic-cell rescaling (2020)

Like Berendsen barostat, but adds a stochastic term to the rescaling factor so that the local pressure fluctuations are correct for the NPT ensemble. This fixes the canonical-averages problem of Berendsen.

### Andersen / Nosé-Hoover barostat (1980 / 1986)

Add the volume of the box as an extra degree of freedom with potential energy PV and kinetic energy (Q/2)(dV/dt)² with Q being the fictitious mass of the piston. Small Q yields rapid oscillations of the volume; large Q gives slow, overdamped relaxation. The coordinates in the extended system are related to real coordinates by r'_i = V^{-1/3} r_i.

Simulating the microcanonical ensemble in this extended system produces the NPT ensemble in the physical system.

### Monte-Carlo barostat (2002)

Generate a random volume change from a uniform distribution, evaluate the potential energy of the trial system at the new volume, and accept or reject the move with the standard Metropolis Monte-Carlo probability. This directly samples the NPT partition function and produces correct NPT averages.

## Key answers

**Q13. Describe the microcanonical (NVE), canonical (NVT), and isobaric-isothermal (NPT) ensemble.**

NVE: N, V, and E all fixed. The system is isolated. Total energy is conserved exactly. This is what an unconstrained MD simulation samples.<br>
NVT: N and V fixed, energy fluctuates. Temperature T is specified; the system is in thermal contact with a heat bath. Velocities follow the Maxwell-Boltzmann distribution.<br>
NPT: N fixed, both energy and volume fluctuate. Temperature T and pressure P are specified. The system is in contact with both a heat bath and a pressure bath. This ensemble best matches most lab conditions.

**Q14. How is temperature controlled in an MD run? Name at least two options and their advantages/disadvantages.**

Berendsen thermostat: couples the system to a heat bath by scaling velocities so the temperature relaxes exponentially toward T₀ with time constant τ. Pros: simple, easy to implement, smaller discontinuities than direct rescaling, T(t) can fluctuate. Cons: does not generate rigorous canonical averages; artificially suppresses temperature differences between subsystems.<br>
Nosé-Hoover thermostat: adds an extra degree of freedom representing the heat reservoir, simulating an extended system microcanonically to produce the canonical ensemble in the physical system. Pros: generates correct canonical averages, one of the most accurate methods. Cons: more complex to understand and implement; can be non-ergodic or oscillatory without Nosé-Hoover chains.

**Q15. For computing dynamic properties from an MD run, name one good and bad choice of a thermostat, and explain your choice.**

Good: Nosé-Hoover. It is deterministic and does not randomly modify velocities, so the velocity autocorrelation functions are physical. Green-Kubo integrals over these functions give correct transport coefficients.<br>
Bad: Andersen thermostat. It works by randomly reassigning velocities from the Maxwell-Boltzmann distribution at random intervals. This destroys velocity correlations, making any autocorrelation-based dynamic property (diffusion, viscosity) incorrect.

**Q16. How is pressure controlled in an MD run? Name at least two options and their advantages/disadvantages.**

Berendsen barostat: scales the box volume so the pressure relaxes toward P₀ with time constant τ_P; all coordinates scale as r'_i = λ^{1/3} r_i. Pros: simple, easy to implement. Cons: does not produce correct NPT volume fluctuations.<br>
Andersen/Nosé-Hoover barostat: adds the box volume as an extra degree of freedom with fictitious piston mass Q, producing the NPT ensemble in the physical system when the extended system is run microcanonically. Pros: generates correct NPT averages. Cons: more complex, and Q must be chosen carefully to avoid too-rapid or too-slow volume oscillations.
