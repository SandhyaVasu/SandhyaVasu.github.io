---
title: Eighteen Months Inside a Simulation Tool
date: 2026-09-07
category: from-the-lab
tags: [COMETS, floating point]
summary: A year of chasing bugs in COMETS — and what they taught me about floating-point arithmetic, science, and perseverance 
---
{: .fig-right}
![intro](/assets/posts/intro_comet.png)
This tale started with something that should not have been possible.
<br>
I was working on a course project in computational systems biology — a relatively simple simulation of three microorganisms, trying to model butyrate production in a community. The goal was straightforward: run dynamic flux balance analysis, see if the organisms grow, and note what they produce. I was using COMETS, a well-regarded Java-based platform with a Python wrapper, built by the Segrè Lab at Boston University. It had a good reputation. I expected to spend a few hours on setup and then get to the biology.

Instead, I found something that made me sit and stare at my screen for a long time.

When I added the three species to the simulation in the order A, B, C, I got one set of growth trajectories. When I added them as B, C, A — same species, same parameters, same initial conditions, same everything, just a different order at `t = 0` — I got a completely different result. Not a small floating-point rounding difference. Different trajectories. Curves that diverged and went their own ways.

I had not changed the biology. I had only changed the order of the models in the array, and that too at `t = 0`!

This is the story of what I found when I looked into why.

---

## A Quick Word on What COMETS Does

Before I get into the debugging, let me briefly explain what this tool is and why it matters — because it will make this narrative more meaningful.

COMETS (Computation of Microbial Ecosystems in Time and Space) is an excellent platform for simulating microbial communities in space and time. It uses **dynamic flux balance analysis (dFBA)** — a method that takes the constraint-based metabolic models we build for individual organisms and runs them forward through time in a shared environment. Every time step, each organism "decides" what to consume and produce based on what nutrients are available, its own metabolic network, and whatever bounds you've set. The environment updates. You repeat.

This is powerful because you can model how community composition evolves, what nutrients get depleted or accumulated, and how organisms compete or cooperate — all grounded in actual metabolic networks rather than hand-tuned equations.

---

## March 2025: Something Is Very Wrong

The primary problem — the one that stopped me cold — was an `ArrayIndexOutOfBoundsException`. Java's way of telling you that something tried to read a memory location that doesn't exist. I raised it on the COMETS GitHub repository. By adjusting some parameters, I managed to get past it, which at the time felt like a solution: the crash went away, the project could proceed, and I had a deadline.

I did notice, in the course of those runs, that different model arrival orders were producing different growth trajectories. But I didn't investigate it or report it — I noted it, filed it somewhere in the back of my head, and submitted the project report and immersed myself in the next semester of courses.

In September 2025, I revisited my COMETS code and again encountered the order-dependence issue. This time, I raised it as a separate GitHub issue. Their response was quite helpful but couldn't pin down a cause, and I didn't dig deeper into the issue.

---

## December 2025: The Units Rabbit Hole

I came back to COMETS in December, determined to get things working properly. This time, the problem wasn't just the order bug or the `ArrayIndexOutOfBoundsException` error — it was the parameters. Was I setting it correctly?

COMETS uses Michaelis-Menten kinetics to model nutrient uptake. You supply a `Km` value — the half-saturation constant, the nutrient concentration at which the uptake rate is half its maximum. But what **units** does COMETS expect for `Km`?

I could not find clarity in the documentation. I went back and forth from the COMETS paper to their supplementary material to their worked-out examples on the website. Adding to the confusion, published papers using COMETS had used different conventions — mM and M! I became convinced, through a combination of reading and misreading, that the answer was millimolar (mM). This felt right to me — Km values in microbiology are commonly cited in millimolar. With this, I was able to circumvent the error to some extent.

But I was wrong. COMETS expects molar (M). I would not discover this until March 2026. In the meantime, I had built a set of simulations on a foundation that was off by a factor of a thousand.

---
![mess](/assets/posts/mess.png)
## March 2026: Complete Chaos

In March 2026, I had realised my Km units were wrong, corrected them, but was now stuck with the `ArrayIndexOutOfBoundsException`. Was it because the Km was too low that nutrients got depleted? I had no clue. The order dependence issue from a year ago had never been explained. At times, the simulations would keep running forever.

{: .fig-left}
![mess](/assets/posts/comparision.png)
The tool felt broken in ways I couldn't diagnose because the relevant logic was written in Java, with which I was not conversant. My advisor had earlier pointed me to a paper by Andreas Wagner that described a resource-partitioning approach to dFBA — a fundamentally different way of handling nutrient uptake in a community. In COMETS, each species computes how much it can consume as if it were alone in the environment, and over-consumption is corrected after the FBA uptake is done. Wagner's approach partitions nutrients *before* any FBA runs: each species gets a share proportional to its biomass relative to the whole community. I thought this could be causing the issues I was facing. 

I went to my advisor. I told him I couldn't trust the results and I was considering switching to a different dFBA platform entirely. He said, **"Why don't you fix the Java source script?"**

I do not know Java. I had never decompiled a JAR file. The idea of going into the source code of a scientific computing tool and editing it felt impossible for me! But the way he said it made it seem possible. He further asked me to take Claude's help for the same. 

Then began Project Java: an adventurous expedition!

---

## Project Java

The COMETS distribution ships as a compiled JAR file — a bundle of Java bytecode that needs to be decompiled to be read. First up was an extensive hunt to identify the file that contained the dFBA logic. It turned out to be: `FBACell.java`, about a thousand lines long. 

The specific goal was to find where COMETS computed nutrient uptake rates and rewrite it to match Wagner's method. That meant understanding two things: what Wagner's algorithm actually does in mathematical terms, and what the existing COMETS code does — line by line, array by array — so I could make a change I could defend. I explained my case to Claude, and it started making the changes. 

After it was done, I would start at the beginning and begin asking questions. Why did you make this change? What does this line do? What is this variable? What is the Wagner counterpart? Claude would explain. I would not let go of it till I was fully convinced.

This part took weeks. But something unexpected happened along the way: I started understanding Java! Not fluently — I would not go looking for opportunities to write it — but well enough to read it carefully, follow the logic, and catch when something seemed wrong.

And things were wrong.

---

## Attacking the `ArrayIndexOutOfBoundsException` — Finally!

This was the long-standing issue, and it requires understanding something about how COMETS represents metabolites internally.

COMETS maintains two different numbering systems or **index spaces** for metabolites:

| Space | What it numbers | Typical size |
|---|---|---|
| Global media index | Every metabolite in the world at this location | ~500–1000 |
| Model-local exchange index | Metabolites this *specific model* can exchange | ~100–200 |

These are different arrays with different lengths. A global index of, say, 400 might correspond to local exchange index 12 for one model — a mapping that COMETS tracks and provides via a method called `getModelMediaIndexes()`.

Confusing the two is catastrophic. In the re-optimisation loop — the part of the code that's supposed to detect and correct over-consumption — the original code did exactly this:

```java
// BUG — k is a global media index, but deltaMedia is model-local:
double newUptake = thisCellMedia[k] * (this.deltaMedia[l2][k] / totUptake);
int[] modelMediaIndexes = this.world.getModelMediaIndexes(this.x, this.y, l2);
int kIndexInModel = ArrayUtils.indexOf(modelMediaIndexes, k);
```

The translation from global to local index (`kIndexInModel`) was computed — but **three lines too late**. The line that actually used it (`this.deltaMedia[l2][k]`) had already run with the global index `k` instead.

Two things can happen with this bug:

- If `k` happens to be larger than the local array length, you get an `ArrayIndexOutOfBoundsException`. This is the crash I kept seeing.
- If `k` happens to fall within the local array bounds — which it will, silently, whenever the global index happens to be less than ~150 — **there is no crash**. COMETS reads the flux of a completely unrelated metabolite and uses it to rescale the uptake bound. No warning. No error. Just wrong numbers.

The second case is worse than the first. A crash tells you something is broken. A silent wrong answer does not.

The fix was to move the translation to happen first, then use the translated index:

```java
// FIX — translate first, use the translated index:
int[] modelMediaIndexes = this.world.getModelMediaIndexes(this.x, this.y, l2);
int kIndexInModel = ArrayUtils.indexOf(modelMediaIndexes, k);
double newUptake = thisCellMedia[k] * (this.deltaMedia[l2][kIndexInModel] / totUptake);
```

## A Silent Wrong Answer: A Sorting Bug
Now, another discovery was that the metabolites in the environment and the dilution rates were sorted alphabetically upon user input. However, the refresh rates did not get sorted, resulting in refresh rates landing on the wrong metabolites. This was addressed by sorting the metabolites in my simulation script. 

---

### The Clamp That Creates or Destroys Mass

When COMETS updates the nutrient medium after all models have run, it calls `changeModelMedia()` once per model. That function does two things:

```java
media[k] += delta;                     // add this model's consumption/secretion
if (media[k] < 0) media[k] = 0;       // clamp to zero if negative
```

This clamp — sensible in isolation (removes floating point negative noise) — becomes pathological when applied sequentially across models. Consider a simple example: a metabolite at 5 mmol, with model A consuming 8 mmol and model B secreting 4 mmol.

| Processing order | Result |
|---|---|
| A first, then B | `5 − 8 = −3` → clamp → `0`, then `0 + 4 = **4 mmol**` |
| B first, then A | `5 + 4 = 9`, then `9 − 8 = **1 mmol**` |

The final medium concentration is 4 in one order and 1 in the other. Three millimoles appeared from nowhere in the first case. This is not floating-point noise — this is a straightforward violation of mass conservation, and it depends entirely on which model gets processed first.

**The fix:** Before any call to `changeModelMedia`, combine all models' deltas for each metabolite into a single net change, and then pass it to the clamp. Mass is conserved.

These issues, once fixed, meant the tool ran without crashing. At least now, will the order dependency vanish? I was even using pFBA all along. With all this in place, the magnitude of difference between alternate orders reduced but did not disappear.

## The power of floating point arithmetic!

This one was more subtle, and it touched something fundamental about how computers do arithmetic.

### Why 1 + 2 + 3 ≠ 3 + 2 + 1 (sometimes)

Most people assume floating-point addition is like regular addition — the order doesn't matter, you get the same answer. This is true for two numbers. With three or more, it breaks down.

Floating-point numbers are stored with finite precision — roughly 16 significant decimal digits for a 64-bit `double`. When you add two numbers of very different magnitudes, the smaller one loses precision. The result depends on which number you add first, because different groupings round differently.

Mathematically: `(A + B) + C = A + (B + C)`. But in floating-point arithmetic: **sometimes not**. The difference is tiny — typically the last bit of the 64-bit representation, around `1e-16` relative — but it is real.

COMETS accumulates quantities across models by looping through them in the order they appear in the layout array. And models appear in layout-array order because they were added in that order. So the order you add species determines the order of accumulation — and for communities of two or more species, different accumulation orders give different results. (Note, even in two species (A, B) in the media update step, there are three terms: media conc., effect of A, effect of B — causing non-associative arithmetic).

For my simulations, across the time steps, this played a key role in deciding how the solver steered the system of linear equations and landed in an optimum.

Many metabolic models — especially AGORA2 models, which are common in human gut microbiome research — are **degenerate**: they have multiple optimal solutions with identical objective values. The LP solver is free to return any of them. When one run and another differ by a single last bit in a constraint coefficient, the solver may pick a different vertex on the same flat optimal face. Same growth rate, different flux distribution. Different secretion profile. Different nutrient environment next step. One timestep later, the trajectories have diverged.

I found this empirically: in a pair of species where one model had an unconstrained demand reaction for sodium (`DM_NA1`, an ATP-neutral loop), a last-bit difference in the medium vector was enough to switch that loop on or off — at zero cost to growth, but with downstream effects on the sodium available to the other species. The two orders agreed to full precision through cycle 2, diverged at cycle 3, and were on clearly different paths by cycle 5.

**The fix:** Sort models by file name before performing any addition — a key that doesn't depend on arrival order — and walk that fixed order during summation. The sort is computed once at the top of the timestep, before any biomass values are mutated, and reused everywhere.


---

## Two working versions

The Wagner implementation was the primary mission, but the fixes above — the index bug, the refresh sorting, the clamp, the floating-point accumulation order — applied equally to the original uptake logic. So I ended up with two working versions of COMETS: original and Wagner variant, both fully functional and completely order-independent.

---

## Final verdict

I ran a permutation sweep: 69 communities of two to seven species, 1,960 simulations, 1,891 comparisons between different arrival orders. Divergence was exactly zero in every comparison, across biomass, medium composition, and per-reaction fluxes, at every cycle.

---

## Reflections

{: .fig-right}
![intro](/assets/posts/victory.png)
Overall, this was an intense exercise that helped me cultivate patience and perseverance. The process would be so addictive that at times, I would helplessly be up late at night wrangling with the problem. Many times, I would give up hope and feel totally lost. Further, I am sure I would have had many subtler learnings from this. My sincere gratitude to my advisor, Prof. Karthik, whose one line, **"Why don't you fix the Java source script?"**, kept me ignited throughout this endeavour. 



*Sandhya Vasu — September 2026*

*COMETS: [segrelab.org/comets](https://www.segrelab.org/comets/) | Wagner reference implementation: [Wagner's paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC9542400/)*
