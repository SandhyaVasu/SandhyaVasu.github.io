---
title: When the Order You Add Species Changes Your Science
date: 2026-09-07
category: from-the-lab
tags: [COMETS, ]
summary: A year of chasing bugs in COMETS — and what they taught me about floating-point arithmetic, open-source science, and why I love a tool enough to fix it myself
---

It started with something that should not have been possible.

I was working on a course project in computational systems biology — a relatively simple simulation of three microorganisms, trying to model butyrate production in a community. The goal was straightforward: run dynamic flux balance analysis, see if the organisms grow, see what they produce. I was using COMETS, a well-regarded Java-based platform with a Python wrapper, built by the Segre Lab at Boston University. It had a good reputation. I expected to spend a few hours on setup and then get to the biology.

Instead I found something that made me sit and stare at my screen for a long time.

When I added the three species to the simulation in the order A, B, C, I got one set of growth trajectories. When I added them as B, C, A — same species, same parameters, same initial conditions, same everything, just a different order at `t = 0` — I got a completely different result. Not a small floating-point rounding difference. Different trajectories. Curves that diverged and went their own ways.

I had not changed the biology. I had only changed the order of three function calls in a setup script.

This is the story of what I found when I looked into why.

---

## A Quick Word on What COMETS Does

Before I get into the debugging, let me briefly explain what this tool is and why it matters — because it will make the bugs more meaningful.

COMETS (COnstrainedMinimisation of Exchange-Reactions via Triangulated System) is a platform for simulating microbial communities in space and time. It uses **dynamic flux balance analysis (dFBA)** — a method that takes the constraint-based metabolic models we build for individual organisms and runs them forward through time in a shared environment. Every time step, each organism "decides" what to consume and produce based on what nutrients are available, its own metabolic network, and whatever bounds you've set. The environment updates. You repeat.

This is powerful because you can model how community composition evolves, what nutrients get depleted or accumulated, and how organisms compete or cooperate — all grounded in actual metabolic networks rather than hand-tuned equations.

The specific question I study is **priority effects**: whether the order in which species arrive in a community changes its final composition. It is, frankly, a terrible kind of research project to run on a tool that turned out to be sensitive to model arrival order for reasons having nothing to do with biology. But that discovery came later.

---

## March 2025: Something Is Very Wrong

Back to the anomaly. I reported it as an issue on the COMETS GitHub repository and described what I was seeing. I also filed a second issue I'd noticed: occasionally the simulation would crash with an `ArrayIndexOutOfBoundsException` — Java's way of telling you that something tried to read a memory location that doesn't exist.

The response was helpful but inconclusive, and I had a course project to submit. I documented what I'd seen, moved on, and filed it away as "something to return to."

---

## December 2025: The Units Rabbit Hole

I came back to COMETS in December, determined to get things working properly. This time the problem wasn't the order bug — it was the parameters.

COMETS uses Michaelis-Menten kinetics to model nutrient uptake. You supply a `Km` value — the half-saturation constant, the nutrient concentration at which uptake rate is half its maximum. But what **units** does COMETS expect for `Km`?

The documentation was not explicit about this. Published papers using COMETS had used different conventions. I became convinced, through a combination of reading and misreading, that the answer was millimolar (mM). This felt right to me — Km values in microbiology are commonly cited in millimolar.

I was wrong. COMETS expects molar (M). I would not discover this until March 2026. In the meantime I had built a set of simulations on a foundation that was off by a factor of a thousand.

> **A note on documentation:** I want to be careful here, because it's genuinely easy to get this wrong. The COMETS team has built something remarkable and has documented it considerably. But the gap between a working Python wrapper and a complete understanding of what's happening in the Java backend is real, and for Km units specifically, clarity is hard to find without going back to primary sources. This is not a criticism so much as an honest record of the obstacle.

---

## March 2026: Complete Chaos

By March 2026 I had realised my Km units were wrong, corrected them, and was still getting the `ArrayIndexOutOfBoundsException`. The order-dependence from a year ago had never been explained. The re-optimisation loop — a mechanism inside COMETS that's supposed to prevent over-consumption of nutrients — would occasionally hang indefinitely.

The tool felt broken in ways I couldn't diagnose because the relevant logic was inside Java bytecode I hadn't read.

I went to my advisor. I told him I couldn't trust the results and I was considering switching to a different dFBA platform entirely.

His response was, essentially: before you abandon it, why don't you go into the Java and fix it?

I want to be honest about what I felt when he said that. I do not know Java. I had never decompiled a JAR file. The idea of going into the source code of a scientific computing tool and editing it felt like a thing other people did — people who were primarily software engineers, not biology students with a Python habit.

He said: use Claude.

So I did.

---

## End of May 2026: Into the Source Code

The COMETS distribution ships as a compiled JAR file — a bundle of Java bytecode. To see the source, you have to **decompile** it: run a tool that reads the bytecode and reconstructs (approximately) the Java it came from. The result isn't always perfect — decompilers fill in syntax, and sometimes they get it slightly wrong — but it's readable.

The file I needed was `FBACell.java`. It contains the `run()` method — the function that executes one timestep of the dFBA simulation for a single grid cell. Every decision about nutrient uptake, every call to the linear programming solver, every update to biomass and medium concentrations — it's in there. It's also about a thousand lines long.

I started at the beginning and started asking questions. What does this variable mean? What does this line do? What is this index counting? Why is this array this length?

Claude would explain. I would look at the original code and ask: are you sure? Can you trace this back to where the array is defined? What happens if this index goes out of bounds?

This part took weeks, off and on. But something unexpected happened: I started to understand Java. Not fluently — I would not go looking for opportunities to write it — but well enough to read it carefully, follow the logic, and catch when something seemed wrong.

And things were wrong.

---

## A Silent Wrong Answer: The Index Space Problem

This one is more serious, and it requires understanding something about how COMETS represents metabolites internally.

COMETS maintains two different numbering systems for metabolites — what the documentation calls **index spaces**:

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

---

## The Deeper Problem: Order Dependence

The index bug, once fixed, meant the tool ran without crashing. But it was not the source of the strange result I had found back in March 2025.

That one was more subtle, and it touched something fundamental about how computers do arithmetic.

### Why 1 + 2 + 3 ≠ 3 + 2 + 1 (sometimes)

Most people assume floating-point addition is like regular addition — the order doesn't matter, you get the same answer. This is true for two numbers. With three or more, it breaks down.

Floating-point numbers are stored with finite precision — roughly 16 significant decimal digits for a 64-bit `double`. When you add two numbers of very different magnitudes, the smaller one loses precision. The result depends on which number you add first, because different groupings round differently.

Mathematically: `(A + B) + C = A + (B + C)`. But in floating-point arithmetic: **sometimes not**. The difference is tiny — typically the last bit of the 64-bit representation, around `1e-16` relative — but it is real.

COMETS accumulates quantities across models by looping through them in the order they appear in the layout array. And models appear in layout-array order because they were added in that order. So the order you add species determines the order of accumulation — and for communities of three or more species, different accumulation orders give different results.

For most simulations this last-bit difference doesn't matter. For my simulations, it did.

The reason comes down to a property of the linear programs being solved. Many metabolic models — especially AGORA2 models, which are common in human gut microbiome research — are **degenerate**: they have multiple optimal solutions with identical objective values. The LP solver is free to return any of them. When one run and another differ by a single last bit in a constraint coefficient, the solver may pick a different vertex on the same flat optimal face. Same growth rate, different flux distribution. Different secretion profile. Different nutrient environment next step. One timestep later, the trajectories have diverged.

I found this empirically: in a pair of species where one model had an unconstrained demand reaction for sodium (`DM_NA1`, an ATP-neutral loop), a last-bit difference in the medium vector was enough to switch that loop on or off — at zero cost to growth, but with downstream effects on the sodium available to the other species. The two orders agreed to full precision through cycle 2, diverged at cycle 3, and were on clearly different paths by cycle 5.

**The fix (O1):** Sort models by file name before accumulating — a key that doesn't depend on arrival order — and walk that fixed order at every accumulation site. The sort is computed once at the top of the timestep, before any biomass values are mutated, and reused everywhere.

### The Clamp That Creates Mass

The second order-dependence problem is worse because it's not just a last-bit issue — it can produce macroscopically wrong results.

When COMETS updates the nutrient medium after all models have run, it calls `changeModelMedia()` once per model. That function does two things:

```java
media[k] += delta;                     // add this model's consumption/secretion
if (media[k] < 0) media[k] = 0;       // clamp to zero if negative
```

This clamp — sensible in isolation, you can't have negative nutrient concentrations — becomes pathological when applied sequentially across models. Consider a simple example: a metabolite at 5 mmol, with model A consuming 8 mmol and model B secreting 4 mmol.

| Processing order | Result |
|---|---|
| A first, then B | `5 − 8 = −3` → clamp → `0`, then `0 + 4 = **4 mmol**` |
| B first, then A | `5 + 4 = 9`, then `9 − 8 = **1 mmol**` |

The final medium concentration is 4 in one order and 1 in the other. Three millimoles appeared from nowhere in the first case. This is not floating-point noise — this is a straightforward violation of mass conservation, and it depends entirely on which model gets processed first.

**The fix (O2):** Before any call to `changeModelMedia`, combine all models' deltas for each metabolite into a single net change, assign that net change to exactly one canonically chosen "owner" model, and set every other model's delta for that metabolite to zero. Then let the loop run as before — the owner applies the net change and gets clamped once, the others add exactly `0.0` (a no-op in IEEE-754 arithmetic, always). One addition, one clamp, per metabolite, per timestep. Mass is conserved.

> **A note on the Wagner algorithm:** The second build I produced doesn't just fix these bugs — it replaces COMETS's stock uptake logic with the resource-partitioning algorithm described by Andreas Wagner. In stock COMETS, each species computes its uptake bound as if it were alone in the cell: it sees the entire nutrient pool and competes only after the fact. Wagner's method partitions nutrients *before* any FBA runs: each species gets a share proportional to its biomass relative to the whole community. Over-consumption becomes arithmetically impossible (rather than caught and corrected). This is the scientifically cleaner approach for competitive community simulations. The bugs above also exist in that build and are fixed there, but the clamp-creates-mass problem is less severe because the pre-FBA partitioning means the clamp rarely fires. In the stock build, it fires regularly.

---

## Verification

I did not want to take any of this on faith. For each change, I traced the bytecode of the original JAR to confirm the intended semantics of what was being changed. I decompiled the output JARs and read them back to confirm the changes were actually present in the compiled code, not just the source. I ran a permutation sweep: 69 communities of two to seven species, 1,960 simulations, 1,891 comparisons between different arrival orders.

In the unfixed build, 449 of 453 tested pairs showed divergence. Thirteen communities had flux differences above 1 mmol/gDW/h. The largest biomass spread was around 5 × 10⁻⁵ gDW.

In the fixed build: divergence was exactly zero in every comparison, across biomass, medium composition, and per-reaction fluxes, at every cycle.

---

## What I Learned

### About the science

Dynamic FBA on degenerate LP models is fragile. Degeneracy is not a pathological edge case — it is the normal condition of well-constrained metabolic networks. Many reactions can be swapped freely without changing the growth rate. When you perturb a constraint coefficient by a single bit, you can land on a different equivalent optimum with a different flux distribution. If your tool has any arithmetic that changes with model arrival order, that arithmetic will manifest in your results. This has implications for reproducibility that go beyond COMETS.

### About the code

The gap between "the code runs without crashing" and "the code computes what we think it computes" is real and not always obvious. The index space bug — silently reading the flux of an unrelated metabolite — is the kind of bug that produces results that look plausible, pass informal sanity checks, and are simply wrong. You need to understand what the code is doing, not just whether it completes.

### About working with AI

I used Claude extensively throughout this process, and I want to say something honest about how that worked. The process was collaborative in a specific way: Claude would explain, I would verify. Claude would propose a change, I would trace through why the change was semantically equivalent to what the bytecode intended. I would not have been able to do this at anything like the same speed without AI assistance. I also would not have trusted the result if I had not verified each piece myself. The combination — AI for explanation and drafting, human judgment for verification — was more productive than either alone would have been. I also ended up learning a substantial amount of Java, which I did not anticipate.

### About open source

The COMETS team has built something genuinely good. The ability to run spatial, multi-species dFBA simulations with Python-accessible control, grounded in real metabolic networks — there is nothing else quite like it for the kind of work I do. The bugs I found are not evidence of carelessness; they are evidence of the difficulty of scientific software, the complexity of two index spaces that interact in subtle ways, and the genuinely non-obvious properties of floating-point arithmetic.

I did not migrate to another tool. I preferred to understand this one.

---

## A Note on the Journey's Shape

Looking back, this took roughly eighteen months of on-and-off engagement: initial discovery in March 2025, a confusing detour through unit conventions in December 2025, the collapse in March 2026 when everything seemed broken at once, and then a month of focused work from late July to late August 2026 that produced two verified builds.

The non-linear shape of it is, I think, typical of this kind of work. You notice something odd, you don't have the tools to understand it yet, you move on. You come back with more knowledge. You get the wrong answer for a while. Eventually you have enough context that the pieces fit together.

I don't think I would have found the bugs if I hadn't hit the crashes. I don't think I would have understood the crashes without understanding the index spaces. I don't think I would have understood the order-dependence without understanding how LP degeneracy translates a last-bit perturbation into a macroscopic difference. Each piece required the ones before it.

If you're using COMETS for multi-species simulations — particularly priority-effect studies, where you're explicitly varying the order of model arrival — I hope this is useful. The fixed builds, along with the full change documentation, are part of this project's repository.

And if you're staring at a decompiled Java file wondering where to start: it gets easier. Ask questions, verify answers, and don't accept any change you don't understand. The code will eventually tell you what it's doing.

---

*Sandhya Vasu — September 2026*

*COMETS: [segrelab.org/comets](https://www.segrelab.org/comets/) | Wagner et al. reference implementation: metfuncs_aw_pub.py*
