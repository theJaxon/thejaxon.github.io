---
title: "< Hello World />"
date: 2024-11-23T22:33:48+01:00
image:
comments: true
---

I've used [Hugo](https://gohugo.io/) in the past and deployed a sample blog, but that was all.

I published a single post and then forgot about the idea.

This is yet another attempt to write more... I hope so.

---

### How was this blog created ?
```bash
hugo new site thejaxon.github.io && cd thejaxon.github.io

git submodule add https://gitlab.com/gabmus/hugo-ficurinia.git themes/hugo-ficurinia

# Refer to https://gitlab.com/gabmus/hugo-ficurinia#configuration for configuration sample
touch config.toml

# Add blog content
hugo new content content/posts/hello_world.md
```