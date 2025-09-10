# Overview of \$N\$-Body Code Implementation

This document introduces the implementation of direct \$N\$-body calculation codes.

* Note: Equation rendering may break on GitHub.
* In Visual Studio Code, markdown preview displays correctly.

---

## Fundamental Equation

$$
\vec{a}_{i} = \sum_{j \neq i}^{N}{\frac{G m_{j} \vec{r}_{ji}}{\left({r_{ji}^{2} + \epsilon^{2}}\right)^{3 / 2}}}
,\quad\mathrm{where}\,\,
\vec{r}_{ji} \equiv \vec{r}_{j} - \vec{r}_{i}
.
$$

* \$m\_{i}\$, \$\vec{r}*{i}\$, \$\vec{a}*{i}\$ denote the mass, position, and acceleration of particle \$i\$, respectively. \$N\$ is the total number of particles.

  * The particle experiencing the force is denoted as particle \$i\$, while the particle exerting the force is denoted as particle \$j\$.
* \$G\$ is the gravitational constant. In this sample code, we adopt units such that \$G = 1\$.
* \$\epsilon\$ is the gravitational softening parameter. We adopt Plummer softening.

  * A proper understanding of gravitational softening is essential for running \$N\$-body simulations, though details are omitted here.
* In direct methods, the computational cost of gravitational forces scales as \$\mathcal{O}(N^2)\$.

---

## Numerical Integration Schemes

### Leapfrog Method (Second-Order Accuracy)

* Widely used for collisionless \$N\$-body simulations.
* With fixed and shared timesteps, symplecticity is guaranteed, ensuring good energy conservation.

$$
\vec{v}_{i}^{n + 1 / 2} = \vec{v}_{i}^{n - 1 / 2} + \varDelta t \vec{a}_{i}^{n}
,\\
\vec{r}_{i}^{n + 1} = \vec{r}_{i}^{n} + \varDelta t \vec{v}_{i}^{n + 1/2}
.
$$

* Subscript: particle ID, Superscript: timestep index.
* Since velocities are defined at half-step offsets, special predictor and corrector formulas are required at the beginning and end.

$$
\vec{v}_{i}^{1 / 2} = \vec{v}_{i}^{0} + \frac{\varDelta t}{2} \vec{a}_{i}^{0}
,\\
\vec{v}_{i}^{n} = \vec{v}_{i}^{n + 1 / 2} - \frac{\varDelta t}{2} \vec{a}_{i}^{n}
.
$$

* Energy conservation vs. \$\varDelta t\$:

  * <img src="gallery/conservation/fig/leapfrog_scl_err_ene_fp.png" width="600px">  
  * Confirms second-order accuracy.

    * With all single precision (Low: FP32, Mid: FP32), error bottoms out around \$10^{-6}\$.

      * Further reducing \$\varDelta t\$ increases global error, since local error does not decrease but the number of steps grows.
    * With mixed precision (Low: FP32, Mid: FP64), error bottoms out around \$10^{-9}\$.

---

### Hermite Method (Fourth-Order Accuracy)

* Commonly used for collisional \$N\$-body simulations.
* Independent timesteps are possible, but hierarchical timesteps are often adopted to utilize SIMD and multi-core processors effectively.
* Uses a predictor–corrector scheme. Both acceleration and its first time derivative (jerk) are computed, enabling fourth-order accuracy via cubic Hermite interpolation.

$$
\frac{\mathrm{d}\vec{a}_{i}}{\mathrm{d}t} = \sum_{j \neq i}^{N}{G m_j \left[\frac{\vec{v}_{ji}}{\left({r_{ji}^2 + \epsilon^2}\right)^{3 / 2}} - \frac{3 \left(\vec{r}_{ji} \cdot \vec{v}_{ji}\right) \vec{r}_{ji}}{\left({r_{ji}^2 + \epsilon^2}\right)^{5 / 2}}\right]}
.
$$

* The number of active \$i\$-particles per step varies significantly.

* The number of \$j\$-particles is always \$N\$.

* Timesteps are set following the generalized formula of [Nitadori & Makino (2008)](https://doi.org/10.1016/j.newast.2008.01.010).

  * Timesteps are proportional to parameter \$\eta\$.

* Energy conservation vs. \$\eta\$:

  * <img src="gallery/conservation/fig/hermite_scl_err_ene_fp.png" width="600px">  
  * Confirms fourth-order accuracy.

    * With all single precision (Low: FP32, Mid: FP32), error bottoms out around \$10^{-6}\$.

---

### Implementation Examples

| Source Code                                                                   | Description                     |
| ----------------------------------------------------------------------------- | ------------------------------- |
| [cpp/base/0\_base/nbody\_leapfrog2.cpp](/cpp/base/0_base/nbody_leapfrog2.cpp) | Leapfrog method                 |
| [cpp/base/1\_simd/nbody\_leapfrog2.cpp](/cpp/base/1_simd/nbody_leapfrog2.cpp) | Leapfrog method with `omp simd` |
| [cpp/base/0\_base/nbody\_hermite4.cpp](/cpp/base/0_base/nbody_hermite4.cpp)   | Hermite method                  |
| [cpp/base/1\_simd/nbody\_hermite4.cpp](/cpp/base/1_simd/nbody_hermite4.cpp)   | Hermite method with `omp simd`  |

* Note: CPU implementations here are not highly optimized.

---

## Test Problem: Cold Collapse

### Setup

* Particles are uniformly distributed inside a sphere of radius \$R = 1\$.
* System evolves under self-gravity only.
* Total mass \$M = 1\$, with equal-mass particles.

### Background Knowledge for Interpretation

* **Free-fall time** \$t\_\mathrm{ff}\$:

  $$
  t_\mathrm{ff} = \sqrt{\frac{3 \pi}{32 G \rho}}
  = \sqrt{\frac{\pi^{2} R^{3}}{8 G M}}
  .
  $$

  * With \$G = 1\$, \$R = 1\$, \$M = 1\$, we obtain \$t\_\mathrm{ff} \simeq 1.11\$.

* **Virial theorem**:

  * For total kinetic energy \$K\$ and potential energy \$W\$, equilibrium satisfies \$2K + W = 0\$ (scalar virial theorem).

    * Independent of system shape.
  * The virial ratio \$-K / W\$ equals \$1/2\$ in equilibrium.
