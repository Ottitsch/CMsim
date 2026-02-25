
# Describing how atoms interact

The force function is one of the two fundamental ingredients of MD. Given positions, it must return forces. The quality of a simulation is ultimately limited by how accurately the potential energy function captures the real physics. Different systems — noble gases, metals, covalent solids, organic molecules — require fundamentally different approaches.

## Exam questions

- **Q17.** Name at least two two-body potentials and what they are used for.
- **Q18.** What is the general idea of a bond-order potential compared to a regular two-body potential?
- **Q19.** What potential terms are needed for studying molecular systems (bonded and non-bonded terms)?
- **Q20.** Draw the approximate shape of bond, angle, and dihedral terms in a molecular potential energy function.
- **Q21.** Can a bond break in a classical harmonic bond potential? Why/why not?

## The n-body expansion

The potential energy of a system of atoms is, in principle, a function of all coordinates simultaneously. The standard classical approximation expands it as a sum of contributions involving increasing numbers of atoms:

```
E_pot{r_I} = V0
           + Σ_I      V1(r_I)
           + Σ_IJ     V2(r_I, r_J)
           + Σ_IJK    V3(r_I, r_J, r_K)
           + Σ_IJKL   V4(r_I, r_J, r_K, r_L)
           + ...
```

The one-body term V1 is a single-atom energy (e.g. an external field). V2 is a pairwise interaction — the most common approximation. V3 and V4 are three- and four-body terms that capture angular and torsional effects.

The best choice of potential depends on the state and bonding of the system. For non-bonded interactions (noble gases, liquids), a two-body potential is usually sufficient. For bonded interactions (covalent bonds, organic molecules), many-body terms are essential.

## Two-body potentials

### Lennard-Jones potential (1924)

```
V2(R) = 4ε [(σ/R)¹² − (σ/R)⁶]
```

Used for noble gases and as the non-bonded repulsion/dispersion term in many force fields. The (σ/R)⁶ term captures the attractive London dispersion interaction. The (σ/R)¹² term is a computationally convenient but physically approximate repulsive wall. ε is the depth of the potential well; σ is the distance at which the potential crosses zero.

### Buckingham potential (1938)

```
V2(R) = A e^{−BR} − C/R⁶
```

Replaces the LJ repulsive wall with an exponential, which is physically more justified (electron density decays exponentially). More flexible than LJ but slightly harder to implement.

### Morse potential (1929)

```
V2(R) = D_e [(1 − e^{−α(R−R_c)})² − 1]
```

Used for covalent (diatomic) bonds. Has a minimum at R_c with potential energy −D_e. Unlike harmonic bonds, the Morse potential has an asymptote at large R, allowing bonds to dissociate. It describes many diatomic systems well.

### Stillinger-Weber potential (1985)

A combined two- and three-body potential for solid and liquid phases of Si, GaN, 2D materials, and similar covalently bonded solids:

```
V2(R) = Aε [B(σ/R)^p − (σ/R)^q] e^{σ/(R−aσ)}

V3(R_IJ, R_IK, θ_IJK) = λε (cos θ_IJK − cos θ_0)² e^{γσ/(R_IJ−aσ)} e^{γσ/(R_IK−aσ)}
```

The three-body term penalizes deviations from the ideal tetrahedral angle, capturing the directional character of covalent bonds.

### Tersoff bond-order potential (1988)

A two-body potential where the attractive part is modulated by a bond order term b_IJ that depends on the local chemical environment (angle θ_IJK):

```
V2(R_IJ) = f_c(R_IJ) [f_r(R_IJ) + b_IJ f_a(R_IJ)]
```

where f_r(R_IJ) = A e^{−λ₁R_IJ} is the repulsion, f_a(R_IJ) = −B e^{−λ₂R_IJ} is the attraction, and f_c(R_IJ) is a smooth cutoff function. The bond order b_IJ decreases when atom I has many neighbors — a saturated atom forms weaker bonds. Used for Si, C, and related materials.

## Bond-order potentials: the key idea

A regular two-body potential assigns the same bond strength regardless of the chemical environment. This is wrong: a carbon atom with four bonds is saturated and each bond is weaker than a carbon atom with only one bond.

Bond-order potentials fix this by making the depth of the attractive well a function of how many other bonds each atom already has. In the Tersoff potential, b_IJ acts as a modulator: if atom I is bonded to many other atoms K, the term b_IJ is small and the I-J bond is weak. If I has no other neighbors ("naked atom"), b_IJ → 1 and the bond reaches its maximum depth. This is visible in the potential energy curves: the family of curves shifts downward as bond order increases.

## Molecular mechanics

For molecular systems — especially organic molecules in fluid phases — the potential must describe both bonded interactions (the covalent connectivity of the molecule) and non-bonded interactions (van der Waals and electrostatics between atoms not directly bonded):

```
V = V_bonded + V_non-bonded
```

Molecular representations come in three levels of detail:
- **All-atom (AA):** Every atom including hydrogens is explicit.
- **United-atom (UA):** Hydrogens are merged into their heavy atom (e.g. CH₃ becomes one bead). Faster, reasonable accuracy.
- **Coarse-grained (CG):** Multiple atoms form a single bead. Much faster, less chemical detail.

Hybrid approaches also exist, e.g. QM/MM where one region is treated quantum-mechanically and the rest classically.

## The OPLS-AA force field (1996)

A widely used all-atom molecular force field. The total potential is:

**Bonded terms:**

```
V_bond    = Σ_{IJ}    K_R(I,J) [R_IJ − R_{0,IJ}]²

V_angle   = Σ_{IJK}   K_θ(I,J,K) [θ_IJK − θ_{0,IJK}]²

V_dihedral = Σ_{IJKL} K₁/2 [1 + cos(φ_IJKL − φ₁)]
                     + K₂/2 [1 − cos(2φ_IJKL − φ₂)]
                     + K₃/2 [1 + cos(3φ_IJKL − φ₃)]
                     + K₄/2 [1 − cos(4φ_IJKL − φ₄)]
```

**Non-bonded terms (combined vdW and electrostatics):**

```
V_non-bonded = Σ_{I>J} f_IJ (A_IJ/R_IJ¹² − C_IJ/R_IJ⁶ + q_I q_J e²/(4πεk R_IJ))
```

with combining rules A_IJ = √(A_II A_JJ) and C_IJ = √(C_II C_JJ). The scaling factor f_IJ excludes pairs that are bonded (f = 0 for 1-2 and 1-3 pairs) and reduces 1-4 interactions (f = 0.5); all other pairs use f = 1.

## Shape of bonded terms

**Bond (two-body, harmonic):**

The true bond potential is a Morse curve — deep well with an asymptote at large R allowing dissociation. In a molecular force field, the Morse potential is approximated by a Taylor expansion around the equilibrium length R_min, truncated after second order:

```
ΔE_pot = ½ K (l − l₀)²
```

This is a parabola centered at l₀. It is a good approximation near equilibrium but rises symmetrically and infinitely on both sides — a bond modeled this way can never break.

**Angle (three-body, harmonic):**

The bond angle θ_IJK is the angle at atom J between the J-I and J-K bond vectors. The potential is again approximated as a harmonic term around the equilibrium angle θ₀:

```
ΔE_pot ≈ ½ K (θ − θ₀)²
```

A parabola in angle space, rising steeply for deviations from θ₀.

**Dihedral (four-body, periodic cosine series):**

A dihedral angle φ_IJKL is the angle between the I-J-K plane and the J-K-L plane, i.e. the rotation around the J-K bond. Two types:
- **Proper dihedrals:** the four atoms are in a chain I-J-K-L. The potential is a periodic cosine series with multiple minima corresponding to staggered and eclipsed conformations (gauche, anti, etc.).
- **Improper dihedrals:** atom I is the central atom bonded to J, K, L. The dihedral measures out-of-plane bending, used to keep planar groups (e.g. aromatic rings, peptide bonds) flat.

The shape of the dihedral potential is a periodic curve with multiple wells separated by barriers. Unlike bonds and angles, there can be multiple stable minima.

## Can a bond break in a classical harmonic potential?

No. A harmonic bond potential ΔE_pot = ½K(l − l₀)² is a parabola that rises to infinity in both directions. There is no asymptote: the energy cost increases without limit as the bond is stretched, and at no finite distance does the bond energy reach a plateau corresponding to two free atoms.

A Morse potential does allow dissociation, because it has a well at R_c and asymptotes to zero at large R. Once a bond is stretched far enough to pass the inflection point of the Morse curve, the restoring force vanishes and the atoms separate. The harmonic approximation discards this behavior entirely by truncating the Taylor expansion of the Morse potential after second order.

For simulations that require reactivity — proton transfers, combustion, bond-breaking reactions — possible solutions are:
- **Reactive force fields based on bond-order potentials** (e.g. ReaxFF), which use a bond-order formalism to allow bonds to form and break continuously.
- **Machine learning force fields**, which have no enforced functional form and can learn the full reactive energy surface.
- **Ghost atoms**, which work for simple cases like protonation reactions.

## Partial charges and polarizability

Electrostatic interactions in molecular force fields require assigning a partial charge to each atom. There is no unique physical definition of "atomic charge" — it is a model quantity. Common schemes include:
- **Formal charges:** integer charges from Lewis structure electron counting.
- **RESP (Restrained Electrostatic Potential):** partial charges fit to reproduce the quantum-mechanical electrostatic potential around the molecule.
- **Mulliken charges:** partitioned from quantum-mechanical wavefunction coefficients.
- **Bader charges:** partitioned by integrating electron density within zero-flux surfaces.

None of these is uniquely correct; different schemes give different values. Total charge is conserved (Σq = 0 for a neutral molecule) regardless of scheme.

Most force fields use fixed partial charges. In reality, the electron cloud of an atom deforms in response to an electric field — this is **polarizability**. One way to model it is the **Drude particle**: an auxiliary point charge q_D with mass m_D is attached to each atom via a harmonic spring of zero rest length. In the absence of an external field the Drude particle sits on the atom. An external field displaces it, creating an induced dipole. The force constant is the same for all Drude particles; the charge q_D is chosen to reproduce the atomic polarizability.

## Key answers

**Q17. Name at least two two-body potentials and what they are used for.**

Lennard-Jones: V2(R) = 4ε[(σ/R)¹² − (σ/R)⁶]. Used for noble gases and as the non-bonded dispersion/repulsion term in molecular force fields. The R⁻⁶ term captures London dispersion; the R⁻¹² term is a repulsive wall.<br>
Morse potential: V2(R) = D_e[(1 − e^{−α(R−R_c)})² − 1]. Used for covalent (diatomic) bonds. Has a well at R_c, depth D_e, and an asymptote at large R — bonds can dissociate.<br>
Buckingham: V2(R) = A e^{−BR} − C/R⁶. Like LJ but with a physically better-motivated exponential repulsive wall instead of R⁻¹².

**Q18. What is the general idea of a bond-order potential compared to a regular two-body potential?**

A regular two-body potential assigns the same bond strength to every pair, ignoring chemical environment. A bond-order potential modulates the attractive part of the potential by a bond order term b_IJ that decreases as the number of existing bonds increases. An atom with many neighbors (saturated) forms weaker additional bonds; an isolated atom forms the strongest possible bond. This captures the key quantum-mechanical effect that electrons shared among more bonds make each individual bond weaker.

**Q19. What potential terms are needed for studying molecular systems (bonded and non-bonded terms)?**

Bonded terms: harmonic bond stretching (two-body), harmonic angle bending (three-body), and a cosine series for proper dihedral torsions plus improper dihedrals for planarity constraints (four-body).<br>
Non-bonded terms: Lennard-Jones for van der Waals interactions and Coulomb interactions between partial charges, applied to all pairs not directly bonded (with modified weights for 1-4 pairs).

**Q20. Draw the approximate shape of bond, angle, and dihedral terms in a molecular potential energy function.**

Bond: a parabola (upward-opening) with minimum at l₀. Rises steeply and symmetrically on both sides. No asymptote.<br>
Angle: a parabola (upward-opening) with minimum at θ₀. Same symmetric shape in angle space.<br>
Dihedral: a periodic cosine curve with multiple minima (e.g. at 60°, 180°, 300° for a staggered CH₃ group) and barriers between them. The number of minima and their depths depend on the K_n coefficients.

**Q21. Can a bond break in a classical harmonic bond potential? Why/why not?**

No. A harmonic potential ΔE_pot = ½K(l − l₀)² rises to infinity as l increases. There is no asymptote — the energy never reaches a plateau, and the restoring force never disappears. The bond therefore cannot break regardless of how far it is stretched. The true Morse potential does allow dissociation, but the harmonic approximation discards this by truncating the Taylor expansion after second order.
