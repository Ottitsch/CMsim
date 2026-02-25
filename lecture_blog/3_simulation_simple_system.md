
# Running and analyzing a first MD simulation

A minimal MD simulation has a surprisingly small number of moving parts: an initial configuration, a potential energy function, an integrator, and analysis code. But getting it right requires understanding where the computation goes, how sensitive the result is to the time step, and what you can actually measure.

## Exam questions

- **Q6.** What is the most time-consuming step in a naive all-pair evaluation MD? How can it be made faster?
- **Q7.** What happens if the integration time-step is a bit too large? What happens if it is much too large?
- **Q8.** What is a radial distribution function and what information can be obtained from it?
- **Q9.** What is the difference between radial distribution functions and Voronoi tessellation?

## The naive force calculation is O(N²)

For N atoms, each atom interacts with every other atom. That is N(N-1)/2 pairs, which scales as O(N²). For 1000 atoms, that is about 500,000 pair evaluations per time step. For 10,000 atoms, it is 50 million. This is the most time-consuming step in a straightforward MD implementation, and it becomes the bottleneck long before the system size is interesting.

Two standard techniques reduce this cost:

**Cell subdivision** divides the simulation box into a grid of cells, each with edge length slightly larger than the interaction cutoff r_c. For each atom, you only need to check atoms in the same cell and the 26 neighboring cells. Since the number of atoms per cell is roughly constant regardless of N, the total cost drops to O(N).

**Neighbor lists** (Verlet lists) go one step further by precomputing, for each atom, the list of atoms within a slightly extended cutoff r_c + skin. This list is reused for many time steps before being rebuilt. Within each step, only the neighbors in the list are checked. The skin parameter controls the tradeoff: a larger skin means the list is valid for more steps but contains more atoms per check.

Both methods require a cutoff r_c, meaning interactions beyond that distance are neglected. This is fine for short-range potentials like Lennard-Jones but needs special treatment for long-range Coulomb interactions.

## The time step and energy conservation

The time step Δt controls the accuracy of the numerical integration. It must be small enough that the trajectory follows the true trajectory of Newton's equations.

**Slightly too large:** Energy drifts slowly over time. The integration error accumulates, and the simulated system either loses or gains energy systematically. The trajectory is wrong but the simulation does not crash. This is the hardest case to catch because it looks plausible.

**Much too large:** Two atoms can approach each other closely within a single step without the integrator resolving the repulsive wall. The repulsive force is enormous at short range, and the atoms fly apart with huge velocities. This cascades: those atoms hit others, and within a few steps the whole system explodes (velocities diverge, atoms leave the box). This is called a simulation blow-up and is easy to detect because the energy goes to infinity.

A typical safe time step for a Lennard-Jones system in reduced units is around 0.002-0.005 τ. For molecular systems with bonds, the fastest vibrational period (usually C-H stretches) sets the upper bound, and Δt must be a fraction of that period.

## Equilibration before measurement

A simulation started from a crystal lattice or random configuration is not in equilibrium. The kinetic and potential energies are not properly distributed, and measured properties will reflect the initial state rather than the thermodynamic state of interest.

The solution is equilibration: run the simulation for some time before starting to collect data. During equilibration, the system redistributes energy and forgets its initial conditions. You can detect equilibration by monitoring total energy, temperature, or pressure as a function of time and waiting for them to stabilize around a constant value. Only after equilibration should you begin the production run.

## Reproducibility

MD is deterministic given the initial conditions. To reproduce a simulation, you need: the initial positions and velocities, the force field parameters, the integration algorithm and time step, and the random number seed if stochastic elements are involved. Sharing these is essential for reproducibility.

## Static properties from an MD trajectory

Once you have an equilibrated trajectory, you can compute time averages of instantaneous quantities. Static (structural) properties include:

- **Temperature** from mean kinetic energy: T = (2/3N) * <E_k> / k_B
- **Pressure** via the virial theorem
- **Potential energy** and total energy
- **Radial distribution function** (see below)

## The radial distribution function

The radial distribution function g(r) answers: given an atom at the origin, what is the probability of finding another atom at distance r, relative to an ideal gas at the same density?

The definition is:

```
g(r) = n̄(r) / (ρ · V_shell)
```

where n̄(r) is the average number of atoms in a shell of thickness dr at distance r, ρ is the bulk number density, and V_shell = 4πr²dr is the shell volume.

For an ideal gas, g(r) = 1 everywhere. For a liquid:
- g(r) = 0 at small r (atoms cannot overlap)
- The first peak is at approximately the nearest-neighbor distance, indicating the first coordination shell
- Subsequent peaks indicate further shells
- g(r) → 1 at large r (bulk density, no long-range correlations)

For a crystal, g(r) shows sharp delta-function-like peaks at the lattice distances.

Computing g(r) from a simulation requires accumulating a histogram of pairwise distances over many frames, then normalizing by the expected count in each shell for an ideal gas.

## Voronoi tessellation as an alternative structural analysis

The radial distribution function gives a spherically averaged picture of structure. Voronoi tessellation gives a complementary, local, non-averaged view.

In a Voronoi tessellation, space is divided into cells such that each point belongs to the cell of the nearest atom. The Voronoi cell of atom i consists of all points closer to i than to any other atom.

What Voronoi gives that g(r) does not:
- The number of faces of the Voronoi cell is the coordination number of the atom
- The face areas and cell volumes describe local density and packing
- Non-spherical cells indicate anisotropic environments
- It can identify local crystal-like or amorphous environments without assuming spherical symmetry

The key difference: g(r) tells you the distance distribution to neighbors, averaged over all orientations. Voronoi tells you the local geometry around each individual atom, without averaging.

## Key answers

**Q6. What is the most time-consuming step in a naive all-pair evaluation MD? How can it be made faster?**

Force calculation is O(N²) naively because every atom pair must be evaluated. Cell subdivision reduces this to O(N) by only checking atoms in neighboring grid cells within the cutoff. Neighbor lists further reduce per-step cost by precomputing a list of nearby atoms that is reused for many steps.

**Q7. What happens if the integration time-step is a bit too large? What happens if it is much too large?**

Slightly too large: energy drifts slowly. The simulation looks plausible but is systematically wrong.
Much too large: atoms can overlap within one step, the repulsive force becomes enormous, and the system blows up (velocities diverge, atoms leave the box).

**Q8. What is a radial distribution function and what information can be obtained from it?**

g(r) is the probability of finding another atom at distance r relative to what an ideal gas at the same density would give. It reveals coordination shells (peaks), nearest-neighbor distances (position of first peak), and whether long-range order exists (g(r) goes to 1 at large r for a liquid, stays peaked for a crystal).

**Q9. What is the difference between radial distribution functions and Voronoi tessellation?**

g(r) gives a spherically averaged distance distribution: global structure, no directional information.
Voronoi assigns each atom a cell of all points closer to it than to any other atom, giving local per-atom geometry: coordination number (number of faces), cell shape, and local packing, without assuming spherical symmetry.
