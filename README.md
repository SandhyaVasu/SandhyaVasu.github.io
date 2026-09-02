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
tags: [Python, Git]
summary: One or two sentences shown on the homepage card.
---

Your post goes here, in ordinary Markdown — **bold**, `inline code`,
lists, links, headings, code blocks, tables, and so on.
```

That is the whole process. The post appears in the All Posts list
automatically, and the most recent one becomes the featured post at the top of
the homepage.

Notes on the front matter:

| Field | Required | Notes |
| --- | --- | --- |
| `title` | yes | Shown as the page heading and in the list |
| `date` | yes | Should match the date in the filename |
| `tags` | no | Rendered as small tag chips |
| `summary` | no | Falls back to the first paragraph if omitted |

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

Projects are grouped on the homepage under **Ongoing** and **Completed** based
on the `status` field. `link` is optional — without it the title is plain text.

## Editing the homepage and About text

`index.md` holds the About section prose and the short note under the Projects
heading. Both are Markdown.

## Project structure

```
_config.yml            site title, tagline, and Jekyll settings
index.md               homepage — About text, in Markdown
_posts/                blog posts, one Markdown file each
_projects/             projects, one Markdown file each
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
