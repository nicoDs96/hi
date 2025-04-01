---
title: Introducing pyrcodio
layout: post
date: 2025-04-01 19:00
tag: 
- pyrcodio
- python
- name
- generator
- bestemmie
image: assets/images/
headerImage: true
projects: false
hidden: false # don't count this post in blog pagination
description: A blasphemous CLI utility.
category: blog
author: nicods
externalLink: false
---

 <!-- TODO edit image -->
<img class="image" src="{{ site.url }}/assets/images/pyrcodio/Germano_Mosconi.gif" alt="Cover Image"/>

Once upon a time there were some 13-year-old kids bored as hell surfing the internet. They found Germano Mosconi on YouTube and they started to curse God for fun. They never stopped doing it.

NB: Useful for Italian-speaking people. People speaking other languages might not understand the value of this tool.

## Getting Started

I have always been fascinated by how Docker creates amazing container images like "pedantic-ice". After some years spent working with code and countless hours spent debating naming conventions for whatever, here is a solution: `pyrcodio`.

`pyrcodio` is a Python CLI utility that is able to generate names that might result unpleasant for people going to church, but fun for people who don't.

If you are on macOS/Unix, all you need to do to try it out is:

```bash
pip install pyrcodio

pyrcodio
```

To use it while naming Git branches you can:
```bash
git checkout -b $(pyrcodio)
```

<img class="image" src="{{ site.url }}/assets/images/pyrcodio/1.png" alt="[1] Example Git branching"/>

To use it while naming Docker containers you can:

```bash
docker run --name "$(pyrcodio)" -d your-image
```

If you are on Windows, well, pyrcodio...

## Reference
You can find the project on [GitHub](https://github.com/nicoDs96/pyrcodio)