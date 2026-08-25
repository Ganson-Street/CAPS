# Squeeze-Film MOFT Calculation — Method Summary

**Purpose:** summarize the calculation approach for the A349 PXE squeeze-film analysis. Numeric
validation against the published Jang-Khonsari (2008) and Khonsari-Booser (2017) references is covered
separately in `SQUEEZE_FILM_METHOD_VALIDATION.md`.

---

## The problem

Under load, the shaft starts centered (ε = 0) and moves toward the bushing wall as the oil film
resists via squeeze action alone — there is no rotation-driven wedge. Unlike a steady, rotating
journal bearing, this is a transient problem: there is no single equilibrium eccentricity to solve
for. The quantity of interest — MOFT — depends on where the shaft ends up after a fixed available
time, which in turn depends on the entire approach history, not a single-point solve.

Film thickness follows the standard relation $h(\theta) = C(1+\varepsilon\cos\theta)$, giving
$\text{MOFT} = C(1-\varepsilon)$. The calculation therefore reduces to tracking $\varepsilon(t)$ over
the available approach time and reading off its final value.

---

## Approach adopted

We use the method developed by Khonsari and Jang (and adopted in both reference sources) for
finite-length, axially-open squeeze-film bearings: solve the Reynolds equation for the squeeze case,
exploit its linearity in approach rate to avoid an iterative speed search, and march eccentricity
forward in fixed increments, accumulating elapsed time until it reaches the time available.

$$
\frac{1}{R^2}\frac{\partial}{\partial\theta}\!\left(\frac{h^3}{12\mu}\frac{\partial p}{\partial\theta}\right)
+ \frac{\partial}{\partial z}\!\left(\frac{h^3}{12\mu}\frac{\partial p}{\partial z}\right)
= \frac{\partial h}{\partial t}, \qquad p\big|_{z=0,L} = 0
$$

This is algebraically identical to the governing equation in both references (see the validation doc,
§2, for the direct comparison) — only the right-hand side differs from the rotating case, where it's
driven by $U\,\partial h/\partial\theta$ instead of $\partial h/\partial t$.

---

## Why we solve at a unit rate and scale by a ratio

The right-hand side, $\partial h/\partial t = C\dot\varepsilon\cos\theta$, depends on the approach
rate $\dot\varepsilon$ — which is unknown a priori, since it's set by how much rate is needed to
generate enough pressure to support the applied load at the current $\varepsilon$. Because the
equation is linear in $\dot\varepsilon$, pressure (and therefore integrated load) scales exactly with
it. This lets us solve once at $\dot\varepsilon = 1$ and obtain the true rate by a direct ratio, rather
than iterating to convergence at every step:

$$
W_{unit}(\varepsilon) = \iint p\,dA \quad\text{(load if the rate were exactly 1)}, \qquad
\dot\varepsilon = \frac{W}{W_{unit}(\varepsilon)}
$$

$W_{unit}$ is the load-per-unit-rate at the current $\varepsilon$; dividing the actual applied load $W$
by it gives $\dot\varepsilon$ directly. This is the same device both reference sources use in their own
worked examples, expressed there as $\bar W(\varepsilon) = W/(RLK_{sq})$.

---

## Time-integration loop

Eccentricity is advanced in fixed increments $\Delta\varepsilon$; each increment's real-time duration
follows from the rate just computed, since $\Delta t = \Delta\varepsilon/\dot\varepsilon$. The process
repeats until accumulated time reaches the available approach time $T = 30/\text{RPM}$:

1. At the current $\varepsilon$, solve for pressure and compute $\dot\varepsilon$ as above.
2. Convert that rate into a time step: $\Delta t = \Delta\varepsilon / \dot\varepsilon$.
3. Advance $\varepsilon$ by the fixed step, and add $\Delta t$ to the running total of elapsed time.
4. Repeat from step 1 at the new $\varepsilon$, until the running total reaches $T$.

$\dot\varepsilon$ is re-evaluated at every step rather than assumed constant. Physically,
$\dot\varepsilon$ falls sharply as $\varepsilon$ increases — the film thins, resistance rises — so a
fixed $\Delta\varepsilon$ corresponds to short, fast-elapsing steps early on and long, slow-elapsing
steps as the shaft approaches the wall. Nearly all of the available time ends up consumed in the final
portion of the march. This step size has been checked for convergence — refining it further changes
the resulting MOFT by well under 1%.

The process stops once $t \geq T$, and the corresponding $\varepsilon$ gives the final MOFT via the
relation from the first section.

---

## Validation

This method — governing equation, linearity-based rate solve, and time-march — has been checked
directly against the load-capacity and full transient-time results published in both reference
sources, to within 6–17%, consistent with ordinary mesh and cavitation-model differences. Full
numeric comparison is in `SQUEEZE_FILM_METHOD_VALIDATION.md`.
