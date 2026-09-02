---
title: "Automating the Boring Parts: My First CLI Tool"
date: 2026-08-14
tags: [Python]
summary: A walkthrough of building a small command-line utility to batch-rename
  files — and what I learned about argument parsing along the way.
---

I had two hundred photos named `IMG_4471.JPG` and a folder structure that made
no sense. Renaming them by hand was not going to happen, so I wrote a tool.

## Starting with the smallest thing that works

The first version was six lines and did exactly one job:

```python
import os

for i, name in enumerate(sorted(os.listdir("."))):
    ext = os.path.splitext(name)[1]
    os.rename(name, f"photo_{i:03d}{ext}")
```

It worked, and it was also a little terrifying — no confirmation, no undo, and
it happily renamed the script itself on the first run.

## What argument parsing actually buys you

My instinct was to read `sys.argv` directly. That falls apart the moment you
want more than one option. `argparse` handles the tedious parts:

```python
parser = argparse.ArgumentParser(description="Batch-rename files.")
parser.add_argument("pattern", help="naming pattern, e.g. photo_{n}")
parser.add_argument("--dry-run", action="store_true",
                    help="print the renames without doing them")
```

The `--dry-run` flag turned out to matter more than anything else. Being able
to see what *would* happen before it happens is the difference between a tool
you trust and one you run with your eyes closed.

## The lesson

The interesting part of a small tool isn't the logic — mine is still basically
those six lines. It's the guardrails around it: the dry run, the confirmation
prompt, refusing to touch files outside the target directory.

Next time I reach for a shell one-liner three times in a week, that's my signal
to write the tool properly.
