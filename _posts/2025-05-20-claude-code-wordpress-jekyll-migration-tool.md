---
layout: post
title: "Using Claude Code to Build a WordPress to Jekyll Migration Tool in Record Time"
date: 2025-05-20
categories: [tools, automation]
tags: [claude-code, anthropic, python, wordpress, jekyll, migration, cli]
author: "Daniel Streefkerk"
excerpt: "How I leveraged Claude and Claude Code CLI to rapidly develop a bespoke WordPress to Jekyll migration tool, turning what might have been days of work into a few hours of iterative development."
---

I recently launched a new blog on GitHub Pages with Jekyll. I'd run my previous sysadmin-focused blog on various flavours of self-hosted/SaaS WordPress, and I was under the impression that the old blog was long gone as I'd cancelled the WordPress subscription years ago. I was a bit sad that I'd neglected to export a copy of the content, just in case there was anything that could still be of use today.

Just a few days ago, I discovered that the old blog was still online. My first action thereafter was to run an export, which resulted in an emailed link to a 1.5Mb XML file. Not very useful for ingestion into Jekyll/GitHub Pages.

Rather than messing around with [long since abandoned migration tools](https://github.com/sahin/wordpress-migrate-tool), I turned to a combination of [Claude Chat](https://claude.ai/) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code/cli-usage) (Anthropic's new coding assistant CLI) to help develop a bespoke solution in record time.

## The Challenge

WordPress.com provides a 'WordPress eXtended RSS' file that contains all your blog posts/categories/tags/comments, but translating that XML-encoded content to Jekyll's markdown format requires quite a bit of massaging:

- Extracting the encoded content from the HTML
- Converting HTML to proper Markdown
- Setting up appropriate YAML frontmatter with categories and tags
- Handling code blocks and syntax highlighting properly
- Managing embedded content like GitHub gists
- Creating appropriate file naming conventions
- Removing image links (I'm trying to avoid using images on my new blog, it'll just make it simpler and faster to brain-dump useful knowledge if I don't have to think about capturing/editing/uploading/hosting/captioning images)

## Enter Claude Code

I described to Claude exactly what I needed, and it helped me create a complete Python script to handle the migration. I didn't have to teach Claude about WordPress XML exports or Jekyll's format - it already understood both systems well enough to deliver a working script at the first attempt.

My approach was simple:

1. I provided a sample WordPress export XML snippet
2. I described my desired Jekyll output format and naming conventions
3. I asked Claude to generate a Python script that would handle the conversion

Within minutes, I had a working first draft that parsed WordPress export files and generated Jekyll posts. Through a few iterations of refinement, we added:

- A customisable prompt template to guide the content conversion
- Quality checking features to ensure proper formatting
- Support for filtering by categories and tags
- Rudimentary error handling and logging
- Command-line arguments for flexible usage

## The Result

The final script is surprisingly robust for something developed in just a few hours. Here's a simplified usage example:

```bash
python wp_to_jekyll.py --input wordpress_export.xml --output _posts
```

You can see [the full script here](https://gist.github.com/dstreefkerk/06af5de795ca39f8cc8ae8eb38251e2b), but what's more interesting is how quickly I was able to develop something customised to my exact needs. The script leverages Claude itself to handle the nuanced conversion of each post's content, ensuring high-quality migration that respects my specific formatting preferences.

What would have been days of coding, testing, and refining was compressed into a few hours of collaborative development with Claude. The tool even handles custom prompt templates, so I can fine-tune exactly how I want the migration to work:

```bash
python wp_to_jekyll.py --input wordpress_export.xml --output _posts --prompt-file jekyll-migration-prompt.md
```

## Reflections on AI-Assisted Development

This experience highlighted a few things that I think are worth reflecting on:

1. **Specialised tooling is now trivial to build** - Tasks that used to require days of development can now be accomplished in hours. I also built a similar tool to mass-generate some Microsoft security best practice reference documentation at work that would otherwise have taken approximately 5.5 person-working-weeks of research and mind-numbing boredom to write!

2. **Domain knowledge matters less** - I didn't need to know the details of WordPress XML export format or Jekyll's expectations, as Claude already understood these systems.

3. **Iteration is lightning-fast** - Each refinement took minutes instead of hours, as I could simply describe what wasn't working or what I wanted to add. Of course there are the occasional missteps and issues, but they're becoming less frequent as LLM models advance. You can leverage source control tools to enable quick rollback to known-good code versions.

4. **The human role shifts to specification and supervision** - My job became precisely describing what I wanted rather than figuring out how to implement it.

The code isn't perfect (is code ever perfect?), but it's efficient enough, does exactly what I need, and took a fraction of the time I'd have spent building it manually. It's a bit average in places, but it does the job for my specific migration needs.

It's not often that new tools fundamentally change my approach to solving problems, but LLMs have done exactly that. For bespoke tooling needs, my first instinct is now "let's get Claude to help build this" rather than either settling for an imperfect existing solution or blocking out days for custom development.
