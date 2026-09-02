---
title: The Git Commands I Actually Use Every Day
date: 2026-07-09
tags: [Git]
summary: Not a cheat sheet — just an honest account of the commands that show up
  in my terminal daily and why they stuck.
---

There are a lot of Git cheat sheets. Most of them list commands I have never
typed. This is the short list that actually survived contact with my daily work.

## The ones I run without thinking

| Command | Why it stuck |
| --- | --- |
| `git status` | Reflex. I run it between almost every other command. |
| `git add -p` | Reviews my own change hunk by hunk before it's staged. |
| `git log --oneline -10` | Enough history to orient myself, no pager. |
| `git diff --staged` | Last look at what's about to become a commit. |

## The one that changed how I work

`git add -p` was the big one. Staging a whole file is easy, but it means you
commit things you never actually looked at. Going hunk by hunk forces a review
pass on my own work, and it catches the stray `print()` almost every time.

## The escape hatches

I used to be scared of undoing things. Two commands fixed that:

- `git restore <file>` — throw away unstaged changes to one file
- `git reset --soft HEAD~1` — undo the last commit but keep the work staged

That second one is how I fix a bad commit message or split a commit that turned
out to be doing two things at once.

> Knowing you can undo something is what makes you willing to try it.

## What I still look up every time

Rebasing anything non-trivial. I read the docs, do it slowly, and make a backup
branch first. I don't think that will ever become muscle memory, and I've made
peace with it.
