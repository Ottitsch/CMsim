
# Transferring parameters, cutting off interactions, and handling long-range electrostatics

Chapter 7 has three distinct topics. First, how to reuse force field parameters across molecules rather than reparametrizing from scratch every time. Second, how to handle the fact that pair potentials extend to infinity but we can only compute a finite number of interactions. Third, why Coulomb interactions are a special problem that cannot be solved by a simple cutoff, and how Ewald summation addresses this.

## Exam questions

- **Q22.** How can we transfer classical force field parameters from one molecule to another?
- **Q23.** To which term in the potential energy function does a cutoff apply?
- **Q24.** Name a few options for potential cutoffs. What are their advantages/disadvantages?
- **Q25.** What is Ewald summation used for, and what is its (very general, no details needed) concept?
- **Q26.** To what data are force field parameters usually fit to?

## Transferability of force fields

Force fields range from highly specialized (one force field per molecule species) to very general (one force field covering the whole periodic table). A good general force field can often outperform a poor specialized one.

The central question is: how do we reuse parameters from known molecules when we encounter a new one?

### The atom type concept

Rather than assigning unique parameters to every atom in every molecule, atoms with similar chemical environments are grouped into **atom types** that share the same parameters. The key insight is that a CH₃ group in methanol and a CH₃ group in ethanol behave almost identically — there is no need to parametrize them separately.

**Specialized force fields** assign distinct atom types to every chemically distinguishable atom. For three alcohol molecules (methanol, ethanol, propanol), this results in 18 atom types, 15 bond terms, 21 angle terms, and 13 dihedral terms — a lot of parameters to fit.

**Specialized + atom types** collapses some of that: atoms with the same local environment (e.g. all terminal CH₃ hydrogens) get the same type. The same three molecules need only 14 atom types, 14 bond terms, 19 angle terms, 13 dihedral terms.

**General + atom types** collapses further: H atoms in similar environments across all molecules share a type (e.g. H:4 for all aliphatic hydrogens). The same three molecules need only 5 atom types, 7 bond terms, 12 angle terms, 11 dihedral terms.

With a general force field, a new long-chain alkanol or a cyclic ether introduces **zero new atom types** (long chain) or at most **1–2 new terms** (cyclic geometry) — most parameters transfer directly from the existing library.

### How to handle missing parameters

When a genuinely new parameter is needed, two approaches are used:

1. **Guess parameters** from chemically similar existing ones. For example, a new dihedral V_{H4-C3-O2-H1} can be approximated as V_{H4-C3-O2-C3} if no exact match exists.

2. **Derive parameters from atomic properties:**
   - Reference bond lengths: sum of van der Waals radii, corrected for bond order and electronegativity.
   - Bond force constants: proportional to the product of effective atomic charges and inversely proportional to the cube of the equilibrium distance (Badger's rule).
   - Effective atomic charges: usually fit on diatomic molecules.

### General force fields

General force fields use a decision tree to assign atom types. For carbon, the tree branches by hybridization (sp³, sp², sp), then by chemical environment (next to heteroatom, next to carbons, etc.), yielding types like CH₄, −CH₃, −CH₂−, −CH−. Examples of widely used general force fields: OPLS-AA, CHARMM General Force Field (CGenFF), General AMBER Force Field (GAFF), Merck Molecular Force Field (MMFF), GROMOS.

## Potential cutoffs

### Why cutoffs are needed

Most pair potentials decay quickly with distance (short-sighted). However, they are non-zero for any finite R (non-bounded support). Computing all N² pairs is expensive (see Chapter 3), and under periodic boundary conditions, without a maximum cutoff, atoms would interact with their own images from arbitrarily distant replicas. The cutoff must be smaller than half the shortest box length.

### Direct cutoff

The simplest approach: set E_pot = 0 for R > R_cut.

**Pros:** Simple.

**Cons:**
- Discontinuity in E_pot at R_cut: energy jumps as atoms cross the cutoff radius.
- Forces undefined: the gradient of a discontinuous function is undefined at the discontinuity. This breaks energy conservation and causes artifacts.

### Shifting

Add E_pot(R_cut) to the entire function so that the potential goes smoothly to zero at R_cut:

```
E_shifted(R) = E_pot(R) − E_pot(R_cut)    for R ≤ R_cut
             = 0                            for R > R_cut
```

**Pros:** E_pot is continuous at R_cut.

**Cons:** Forces are still undefined — the shift makes the energy continuous but the derivative (force) still has a jump because the slope of the potential at R_cut is non-zero.

### Switching

Multiply the potential by a smooth switching function S(R) that equals 1 at R_switch and 0 at R_cut, e.g.:

```
S(R) = e^{−1/t} / (e^{−1/t} + e^{−1/(1−t)})    where t = (R − R_switch) / (R_cut − R_switch)
```

**Pros:** Both E_pot and forces are continuous and well-defined everywhere. This is the standard approach for production MD.

**Cons:** The potential is modified in the range [R_switch, R_cut], introducing a small systematic error in that region.

The cutoff applies to **non-bonded terms** (vdW and electrostatics). Bonded terms (bond, angle, dihedral) are only evaluated between atoms connected by the molecular topology and do not need a cutoff.

## Ewald summation

### The problem with Coulomb cutoffs

Van der Waals interactions decay as R⁻⁶, so they are effectively zero beyond a few nanometers. A cutoff introduces only a small, controllable error. Coulomb interactions decay as R⁻¹ — they are long-range and a simple cutoff introduces large, unacceptable errors. This is especially true under periodic boundary conditions, where truncating the Coulomb sum at any finite radius breaks charge neutrality in a way that accumulates over many replicas.

### The Ewald idea

Ewald summation (originally 1921, for computing Madelung constants of ionic crystals) splits the Coulomb potential into two parts:

```
V_Coulomb = V_short-range(R) + V_long-range(R)
```

where the short-range part converges rapidly in real space and the long-range part converges rapidly in reciprocal (Fourier) space.

The full sum over all periodic images is:

```
V_Coulomb = (1/2) Σ_n Σ_i Σ_j  q_i q_j / (4πε₀ |r_ij + n|)
```

where n = (n_x L, n_y L, n_z L) runs over all periodic images (n = 0 is the central box). This sum converges very slowly and is conditionally convergent (positive and negative terms both diverge individually).

### The splitting trick

Split 1/r using a function f(r):

```
1/r = f(r)/r + (1 − f(r))/r
```

choosing f(r) such that f(r)/r decays quickly in real space and (1−f(r))/r decays quickly in Fourier space.

The physical picture: place a Gaussian neutralizing charge distribution of opposite sign around each point charge. This screens the long-range part locally, making the remaining interaction short-ranged. Then add back the Gaussian contributions in Fourier space.

Assuming Gaussian neutralizing distributions ρ_i(r) = q_i (α³/π^{3/2}) e^{−α²r²}:

**Short-range (real space) term:**

```
V_short-range = (1/2) Σ_n Σ_i Σ_j  q_i q_j erfc(α|r_ij + n|) / (4πε₀ |r_ij + n|)
```

where erfc(x) = (2/π) ∫_x^∞ e^{−t²} dt is the complementary error function. This decays rapidly — after a modest cutoff the contribution is negligible.

- Large α (narrow Gaussian): faster convergence in real space.
- Small α (wide Gaussian): slower convergence in real space.

**Long-range (reciprocal space) term:**

```
V_long-range = (1/2) Σ_{k≠0} Σ_i Σ_j  q_i q_j / (πL³) · (4π²/k²) e^{−k²/(4α²)} cos(k · r_ij)
```

with reciprocal vectors k = 2πn/L. This converges rapidly in Fourier space.

- Large α: slower convergence in reciprocal space.
- Small α: faster convergence in reciprocal space.
- Default choice: α ≈ 5/L with 100–200 reciprocal vectors k.

**Correction term** (subtracts the interaction of each Gaussian with itself):

```
V_corr = −(α/√π) Σ_{k≠0} q_k² / (4πε₀)
```

If the surrounding medium is vacuum (not conducting), an additional correction for the medium permittivity is needed.

### Particle-Mesh Ewald (PME)

Plain Ewald summation scales as N². The faster **Particle-Mesh Ewald (PME)** scales as N log N:

- The real-space part is the same as above.
- The reciprocal-space part uses the **Fast Fourier Transform (FFT)** instead of a direct sum.
- Since FFT requires discrete data, atomic point charges are interpolated onto a regular grid (a continuous charge distribution on the mesh), and the FFT is applied to that grid.

PME is the standard method for long-range electrostatics in modern MD codes.

## Quantum mechanics as reference data for force field fitting

Force field parameters are fit to quantum-mechanical (QM) calculations and/or experimental data. The chapter includes a concise QM overview to explain what these reference calculations actually are.

### Historical foundations

- **Planck (1900):** Energy comes in quanta: E = hν = ℏω.
- **Einstein (1905):** Light consists of photons with E = pc; wave number ω = ck.
- **Bohr (1913):** Electrons occupy quantized orbits around the nucleus.
- **de Broglie (1924):** Wave-particle duality extends to matter: E = ℏω and p = ℏk.
- **Schrödinger (1926):** Wave equation: iℏ ∂Ψ/∂t = −(ℏ²/2m) ∇²Ψ + V(x)Ψ, or in time-independent form: EΨ = −(ℏ²/2m) ∇²Ψ + V(x)Ψ.
- **Born (1926):** |Ψ|² is the probability density of finding a particle at a given location.

### Simple solutions

**Particle in a box** (V = 0 inside, V = ∞ outside): solutions are standing waves with quantized energies:

```
Ψ_n(x) = √(2/a) sin(nπx/a),    E_n = n²π²ℏ²/(2ma²)
```

**Hydrogen atom:** The wavefunction separates into a radial part R_nl(r) and an angular part Y_lm(θ,φ) (spherical harmonics): Ψ(r) = R_nl(r) Y_lm(θ,φ). The radial part involves Laguerre polynomials, and the energy levels are E_n = −1/(2n²) in atomic units. The solutions give the familiar atomic orbitals: 1s, 2s, 2px, 2py, 2pz, etc.

### Many-electron systems and Slater determinants

For more than one electron, we build many-electron wavefunctions from one-electron orbitals. A simple product (Hartree product) describes non-interacting particles but does not satisfy the Pauli exclusion principle — electrons are indistinguishable fermions and the wavefunction must be antisymmetric under exchange.

The **Slater determinant** satisfies antisymmetry:

```
Ψ(x₁, x₂) = (1/√2) [Ψ(x₁)Ψ(x₂) − Ψ(x₂)Ψ(x₁)]
```

**Variational principle:** The ground state is the antisymmetric wavefunction with the lowest energy, E = ⟨Ψ|H|Ψ⟩/⟨Ψ|Ψ⟩. We minimize this over all valid antisymmetric Ψ.

### Basis sets

Atomic orbitals are approximated as superpositions of simpler basis functions. The exact hydrogen radial solutions R_nl(r) can be approximated by **Slater-type orbitals (STO)**: R_nl(r) = N_nl P_nl(r) e^{−ζr}, where ζ must be determined. Each STO can itself be approximated by a sum of Gaussians (e.g. 3 Gaussians → STO-3G basis set). Gaussian basis functions make the two-electron integrals that appear in the energy expression analytically tractable.

### Many-electron atoms and molecules

The full molecular Hamiltonian is:

```
H = T_e + V_eN + V_ee + T_N + V_NN
```

where T_e is the kinetic energy of electrons, V_eN is electron-nucleus attraction, V_ee is electron-electron repulsion, T_N is nuclear kinetic energy, and V_NN is nuclear-nuclear repulsion. The Born-Oppenheimer approximation fixes the nuclei and solves only the electronic Schrödinger equation.

Molecular orbitals are linear combinations of atomic orbitals (LCAO): Ψ = Σ_i c_i φ_i.

### Hartree-Fock

Instead of computing all pairwise electron-electron interactions explicitly, Hartree-Fock replaces them with the interaction of each electron with the **mean field** of all other electrons. This turns the many-body problem into a self-consistent single-particle problem:

```
F̂ φ_n = ε_n φ_n
```

where F̂ is the Fock operator (one-electron Hamiltonian + mean-field electron repulsion). Changing to a basis φ_i gives the **Roothaan-Hall equations**:

```
F C_n = E_n S C_n
```

where F_ij = ⟨φ_i|F̂|φ_j⟩ and S_ij = ⟨φ_i|φ_j⟩.

The procedure is iterative: guess a wavefunction → compute mean field → solve HF equation → update wavefunction → repeat until convergence (**self-consistent field, SCF**).

HF calculations provide the reference data (geometries, energies, partial charges via RESP) to which classical force field parameters are fit.

## Key answers

**Q22. How can we transfer classical force field parameters from one molecule to another?**

By using **atom types**: atoms with similar chemical environments across different molecules are assigned the same type and share the same parameters. A specialized force field assigns unique types to every chemically distinct atom (many parameters, high accuracy for the parametrized molecules). A general force field uses coarser groupings (e.g. all sp³ aliphatic CH₃ groups share one type), greatly reducing the number of parameters while maintaining broad transferability. For a genuinely new molecule, most types and parameters transfer directly from the library; the few missing parameters are either guessed from similar existing ones or derived from atomic properties (bond lengths from vdW radii + electronegativity/bond order corrections; force constants from Badger's rule).

**Q23. To which term in the potential energy function does a cutoff apply?**

Non-bonded terms: van der Waals (LJ/Buckingham) and electrostatic (Coulomb) interactions. Bonded terms (bond, angle, dihedral) are defined only between atoms connected by the molecular topology and are always evaluated — they need no cutoff.

**Q24. Name a few options for potential cutoffs. What are their advantages/disadvantages?**

Direct cutoff: set E_pot = 0 beyond R_cut. Simple, but causes a discontinuity in E_pot and undefined forces at R_cut — breaks energy conservation.<br>
Shifting: subtract E_pot(R_cut) from the whole function so the potential reaches zero continuously. E_pot is continuous, but forces are still undefined because the slope is non-zero at R_cut.<br>
Switching: multiply by a smooth function that goes from 1 at R_switch to 0 at R_cut. Both E_pot and forces are continuous and well-defined. Introduces a small systematic modification to the potential in [R_switch, R_cut].

**Q25. What is Ewald summation used for, and what is its (very general, no details needed) concept?**

Ewald summation is used to compute long-range Coulomb interactions in periodic systems without a simple cutoff (which would introduce large errors for R⁻¹ potentials). The concept: split the Coulomb potential into a short-range part, which converges rapidly in real space and is handled with a modest real-space cutoff, and a long-range part, which converges rapidly in reciprocal (Fourier) space and is handled via a Fourier sum. The splitting is achieved by adding Gaussian neutralizing charges around each point charge to screen the short-range interaction, then subtracting them back in Fourier space. The modern variant, Particle-Mesh Ewald (PME), uses FFTs for the reciprocal part and scales as N log N.

**Q26. To what data are force field parameters usually fit to?**

Quantum-mechanical calculations and experimental data. QM data provides reference geometries (equilibrium bond lengths and angles), energies (used to fit torsion potentials and non-bonded parameters), and electrostatic potentials (used to fit partial charges via RESP). Experimental data such as heats of vaporization, densities, and diffusion coefficients are used to validate and refine parameters. For simple atomic parameters, effective charges are often fit on diatomic molecules; reference bond lengths use the sum of vdW radii corrected for bond order and electronegativity.
