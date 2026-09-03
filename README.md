# SandhyaVasu.github.io

My personal site and blog, built with [Jekyll](https://jekyllrb.com) and hosted
on GitHub Pages. All content is written in Markdown; GitHub builds the site
automatically on every push, so there is nothing to install or compile.

## Writing a new post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-a-short-slug.md`:

```markdown
---
title: The Title of the Post
date: 2026-09-02
category: from-the-lab
tags: [Python, Git]
summary: One or two sentences shown on the homepage card.
---

Your post goes here, in ordinary Markdown — **bold**, `inline code`,
lists, links, headings, code blocks, tables, and so on.
```

That is the whole process. The post appears in the Writing section
automatically, newest first.

Notes on the front matter:

| Field | Required | Notes |
| --- | --- | --- |
| `title` | yes | Shown as the page heading and in the list |
| `date` | yes | Should match the date in the filename |
| `category` | no | `from-the-lab` or `beyond-the-lab` |
| `tags` | no | Rendered as small tag chips |
| `summary` | no | Falls back to the first paragraph if omitted |

### The two streams

`category` sorts a post into one of two groups under Writing:

| `category` | Heading | For |
| --- | --- | --- |
| `from-the-lab` | From the lab | research, code, science writing |
| `beyond-the-lab` | Beyond the lab | everything else |

The headings only appear once both streams have at least one post — with a
single stream there is nothing to separate, so the list stays plain. A post
with no `category` falls into a third group, "Other".

### Pictures inside a post

Put the file in `assets/posts/`, then add the side class on the line directly
below the image:

```markdown
![Kanha frees the calf](/assets/posts/kanha-calf.png)
{: .fig-left}
```

`.fig-left` floats the picture to the left with the text wrapping down its
right side; `.fig-right` does the mirror image. Alternate the two down the
post and the pictures zig-zag. Each takes 45% of the column width, and below
600px wide they stack full-width on their own.

The `{: .fig-left}` line is kramdown syntax for "attach this class to the
block above" — it has to sit on the very next line, with a blank line before
the following paragraph. Filenames must not begin with an underscore; Jekyll
ignores those, so the image would 404 on the live site.

## Adding a project

Add a Markdown file to `_projects/`:

```markdown
---
title: Task Tracker CLI
status: completed      # "ongoing" or "completed"
order: 3               # controls the order within its group
tags: [Python]
link: https://github.com/SandhyaVasu/task-tracker
---

A one- or two-sentence description of the project.
```

Projects appear on the homepage as a carousel — one per panel, advancing
automatically, with arrows, dots and keyboard arrow keys. `link` becomes the
"Know more" button; without it the button is omitted.

### Adding a project figure

Each slide has space for an image on the right. Put the file in
`assets/projects/`, named after the project's Markdown file, and it is picked
up automatically:

    _projects/gensemble.md  ->  assets/projects/gensemble.png

To use a different filename, set `image:` in the front matter instead. Either
way the file has to exist, so a wrong name shows the placeholder rather than a
broken image. Publication covers work the same way, in `assets/publications/`.

## Adding a publication

Add a Markdown file to `_publications/`:

```markdown
---
title: Evaluating Metabolic Support in Pairwise Microbial Communities Using MetQuest
authors: Pratyay Sengupta, Sandhya Vasudevan, Karthik Raman
venue: Methods in Molecular Biology
venue_short: Methods Mol Biol
volume: 3006
pages: 195-210
year: 2025
doi: 10.1007/978-1-0716-5080-6_9
link: https://link.springer.com/protocol/10.1007/978-1-0716-5080-6_9
image: metquest-cover.png
order: 1
---

A two-line summary of the work.
```

Every field except `title` is optional and simply omitted from the card when
blank — so a missing page range or year leaves a gap rather than showing
something wrong. Covers go in `assets/publications/`, named after the
Markdown file, and `image:` is only needed to override that.

## Editing the homepage

`index.md` holds the intro prose, the short notes under the Projects and
Publications headings, and the closing quote. All Markdown.

## Project structure

```
_config.yml            site title, tagline, and Jekyll settings
index.md               homepage — intro text, in Markdown
_posts/                blog posts, one Markdown file each
_projects/             projects, one Markdown file each
_publications/         publications, one Markdown file each
assets/posts/          images used inside posts
assets/projects/       project figures
assets/publications/   publication covers
_layouts/
  default.html         the page shell: styles, header, footer, page borders
  home.html            homepage structure
  post.html            individual post pages
```

The three files in `_layouts/` are the design — the CSS and page structure live
there so that everything else can stay Markdown. They rarely need touching.

## Previewing locally (optional)

Not required, since GitHub Pages builds the site for you. But if you want to
see changes before pushing:

```sh
gem install jekyll
jekyll serve
```

Then open <http://localhost:4000>.
