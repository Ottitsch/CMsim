
# Machine-learned force fields

Classical force fields encode decades of chemical intuition in a fixed functional form. That form is fast but limited: it cannot easily capture bond breaking, polarization, or subtle many-body effects. Quantum mechanics is accurate but orders of magnitude too expensive for large systems. Machine learning potentials sit in between — they are trained on QM data and can approach QM accuracy while running at a cost much closer to a classical force field.

## Exam questions

- **Q27.** What is the advantage/promise of machine learning potentials?
- **Q28.** Explain the difference between a fixed and learned descriptor ML potential.
- **Q29.** How are translational and rotational invariance included in ML potentials?
- **Q30.** How is permutational invariance included in ML potentials?

## Why not just use a classical force field or QM?

The true potential energy surface E_pot(x) is complex:
- The number of local minima grows exponentially with N.
- There are multiple transition paths between minima.
- Features exist at different length and energy scales.

Quantum-mechanical calculations are very accurate but far too expensive for the system sizes and timescales MD requires. Classical force fields are cheap but only a crude approximation — they have a fixed functional form that cannot capture the full complexity of the PES. Machine learning can approximate a region of the PES to near-QM accuracy while keeping function evaluations affordable.

## ML as a regression problem

An ML potential is a regression model: given atomic coordinates x as input, predict E_pot(x) as output. The workflow is:

1. **Collect data:** Run QM calculations for many relevant conformations to get a dataset of (coordinates, E_pot) pairs — optionally also atomic forces f = −∇E_pot.
2. **Fit model:** Find the best parameters of the chosen model (e.g. linear regression, kernel methods, neural networks) to reproduce the training data.
3. **Apply model:** For a new configuration, evaluate the model to get E_pot and forces for MD.

## The three invariance problems

A naive ML model that takes raw atomic coordinates as input immediately fails because the learned potential would not be physically invariant under the symmetries of the problem.

### Permutation invariance

Swapping the labels of two identical atoms must leave E_pot unchanged. A model trained on water molecule #1 as the first atom should give the same energy if molecule #1 is relabeled as molecule #2.

**Solution (Assumption 1): Additivity.** Decompose the total energy as a sum of per-atom contributions:

```
E_pot(x) ≈ Σ_i E_pot,i(x)
```

Each atom contributes its own local energy, and the sum is automatically invariant to relabeling because addition is commutative. This makes the model permutation invariant.

### Translation and rotation invariance

Translating or rotating the whole system must leave E_pot unchanged. Raw Cartesian coordinates change under both operations.

**Solution (Assumption 2): Descriptors.** Instead of feeding raw coordinates to the model, first convert each atom's local environment into a descriptor q_i that is invariant to translation and rotation:

```
E_pot,i(x) ≈ E_pot,i(q_i)
```

where q_i encodes only physically meaningful information (distances, angles) about the neighborhood of atom i, not its absolute position or orientation. This makes the model translation and rotation invariant.

## How to design descriptors

Descriptors must be invariant with respect to translation and invariant (or equivariant) with respect to rotation.

### Fixed descriptors

**Two-body (distance-based):** For each atom, bin the distances to its neighbors into a histogram vector — one bin range per neighbor distance. This encodes which atom types are how far away, ignoring angles.

**Three-body (angle-based):** Extend the two-body descriptor with angle information: for each triplet (atom i, neighbor j, neighbor k), record the angle θ_ijk alongside the distances. This captures the local geometry more accurately than distances alone.

**Many-body descriptors** are the standard in practice: they combine a radial part (describing which species are at which distances) with an angular part (based on spherical harmonics or similar angular basis functions) up to a cutoff radius. This captures the full many-body local environment of each atom. Examples from the slides:

- **Atom-centered symmetry functions (ACSFs):** Radial functions × angular functions evaluated over all neighbors up to a cutoff.
- **SOAP (Smooth overlap of atomic positions):** Overlap integral of Gaussian-smoothed neighbor densities.
- **Eigenvalues of the Coulomb matrix:** Sorted eigenvalues of the matrix M_IJ = Z_I Z_J / R_IJ (diagonal: Z_I^2.4 / 2), invariant to permutation and rotation.

**Fixed-descriptor architectures:** GAP (Gaussian process regression on SOAP descriptors), NeuralIL (residual neural network on Bessel descriptors), ANI (neural network on ACSFs).

### Learned descriptors (message passing)

Hand-crafted descriptors rely on expert knowledge about which features matter. An alternative: learn the descriptor from the data itself using **message passing**, where the descriptor of each atom is updated by aggregating information from its neighbors.

Two networks operate in sequence:
- **NN1 (message network):** For each neighbor pair, compute a message vector from their current feature representations.
- **NN2 (update network):** Update each atom's feature vector by aggregating the incoming messages.

This is iterated for several rounds, so each atom's representation accumulates information from increasingly distant neighbors. The final per-atom features are passed to an output network to predict the atomic energy contribution.

### Evolution of learned descriptors

The field has progressed through four generations:

1. **2-body invariant message passing with scalar features:** Single-body messages, basic learned features. Invariant but limited expressiveness.
2. **Multi-body messages including triples (I, J, K) or more:** Incorporates angular information. Can capture more of the local geometry.
3. **2-body equivariant message passing with scalar and vectorial features:** Messages are equivariant vectors (they rotate with the molecule), not just scalars. Can capture directional information while preserving rotational equivariance.
4. **Equivariant + multi-body:** Combines equivariant messages with multi-body interactions. Currently among the best-performing models.

## The long-range problem

All descriptor-based approaches above use a local cutoff: only neighbors within some radius contribute to each atom's descriptor. This means long-range interactions (Coulomb, dispersion tails) are missed entirely.

This is an open problem. Possible approaches:
- **Fixed atomic multipoles:** Assign fixed charges, dipoles, etc. to each atom and handle their long-range interaction classically (e.g. via Ewald summation).
- **ML method for the charge density:** Learn the charge density as a function of local environment, then derive the long-range potential energy from it.
- **Project long-range part onto local descriptors:** Fold long-range contributions into an effective local representation.
- **Ewald-type ML:** Mimic the Ewald split — handle short-range with a local ML model and long-range with a separate ML model in reciprocal space.

None of these has yet become a standard solution.

## Overview of ML potential families

**Neural network models with predefined descriptors:**
ANI, DeepMD, EANN, HDNN, PIP-NN, FI-NN, TensorMol, wACSF, SingleNN, and others.

**Neural network models with learnable descriptors (message passing):**
AIMNet, AIMNet-NSE, DTNN, SchNet, HIP-NN, NequIP, PhyskyNet, PaiNN, SpookyNet, and others.

**Kernel-based models:**
GPR/KRR, GAP, sGDML, SNAP, and others. These fit a kernel regression model (e.g. Gaussian process) on fixed descriptors such as SOAP.

**Linear and polynomial models:**
ACE, ChIMEs, MTP, PIP, and others. Express the potential as a linear combination of basis functions evaluated on the descriptors.

## Useful software

- **scikit-learn** (scikit-learn.org): Comprehensive, well-documented ML library for Python covering the full workflow.
- **TensorFlow** (tensorflow.org): Google's numerical computation library using data flow graphs.
- **PyTorch** (pytorch.org): Meta's alternative to TensorFlow, widely used in the ML potential community.
- **JAX** (github.com/google/jax): Automatic differentiation and JIT compilation for Python/NumPy, aimed at high-performance ML research.

## Challenges and open problems

ML potentials are a rapidly developing field with significant open challenges:

- **No safety net:** Without a fixed functional form, predictions can be arbitrarily wrong outside the training distribution. A model can even predict the wrong sign (attractive instead of repulsive) for interactions it has not seen.
- **Poor generalization:** ML potentials often fail for configurations that differ significantly from the training data.
- **High dependence on training set:** The quality and coverage of the QM training data largely determines the quality of the model.
- **Not off-the-shelf yet:** Results are promising but typically require careful finetuning for each new application.

## Key answers

**Q27. What is the advantage/promise of machine learning potentials?**

ML potentials are trained on quantum-mechanical reference data and can approximate the true potential energy surface to near-QM accuracy, capturing effects (bond breaking, polarization, many-body interactions) that fixed classical force fields cannot. At the same time, once trained, evaluating an ML potential is orders of magnitude cheaper than running a QM calculation, making it feasible to simulate larger systems and longer timescales than QM allows.

**Q28. Explain the difference between a fixed and learned descriptor ML potential.**

A fixed-descriptor ML potential uses a hand-crafted descriptor (e.g. SOAP, ACSFs) that encodes the local atomic environment in a form that is invariant to translation, rotation, and permutation. The ML model (e.g. a neural network or Gaussian process) maps this fixed descriptor to an energy. The descriptor design relies on expert knowledge about which geometric features matter.<br>
A learned-descriptor ML potential instead learns the descriptor from the data using message passing: neural networks iteratively aggregate information from neighboring atoms to build up each atom's feature representation. No manual feature engineering is needed — the model learns which aspects of the local environment are most relevant for predicting the energy.

**Q29. How are translational and rotational invariance included in ML potentials?**

By replacing raw Cartesian coordinates with a descriptor q_i that encodes only physically invariant information about atom i's local environment. Descriptors are built from interatomic distances (invariant to translation and rotation) and angles (invariant to translation and rotation). Many-body descriptors combine a radial part (distances to neighbors by species) with an angular part (based on spherical harmonics) up to a cutoff radius. Since distances and angles do not change under rigid-body translation or rotation, the descriptor — and therefore the predicted energy — is automatically invariant.

**Q30. How is permutational invariance included in ML potentials?**

By assuming that the total energy is a sum of per-atom contributions: E_pot(x) ≈ Σ_i E_pot,i(q_i). Each atom i contributes its own local energy based on its descriptor q_i, and the total energy is their sum. Since addition is commutative, relabeling (permuting) identical atoms leaves the sum unchanged. The model never sees atom indices — only local environment descriptors — so it is automatically invariant to permutation.
