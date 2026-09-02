---
title: Things Nobody Told Me Before My First Code Review
date: 2026-06-18
tags: [Career]
summary: Code reviews aren't about being right. Here's what shifted once I
  understood what they're really for.
---

I submitted my first real pull request expecting a verdict. What I got was
eleven comments, and I spent the first hour reading them as a list of things I
had done wrong.

## It isn't a test you pass

The thing that took longest to sink in: a review is a conversation about the
code, not an assessment of the person who wrote it. Nobody was keeping score.
The comments existed because someone had read my work carefully enough to have
opinions about it — which, I eventually realised, is a form of attention I
should be grateful for.

## Reviews are mostly about the next reader

Roughly half the comments on that first PR weren't about correctness at all.
They were about names, structure, and whether the intent was obvious. At the
time this felt like nitpicking. It isn't. The code works today either way; the
difference shows up in six months when someone — probably me — has to change it.

## What I do differently now

- **Explain the why in the description.** The diff shows what changed. Only I
  know why I chose this approach over the obvious alternative.
- **Review my own diff first.** I catch a surprising number of my own problems
  by reading the change as though someone else wrote it.
- **Ask instead of defending.** "What would you do here?" gets a better outcome
  than justifying my first attempt.
- **Keep PRs small.** A four-file change gets real feedback. A forty-file change
  gets "looks good to me."

## The part that actually mattered

Somewhere in those eleven comments was one that pointed out a real bug — an
off-by-one that would have failed on empty input. If I had been reviewed less
carefully, that would have shipped.

That's the whole point. Not judgement. Just a second pair of eyes.
