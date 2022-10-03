---
layout: default
title: Helpful Posts from Daytonb
---

# Helpful Posts from Daytonb

## Goal of These Pages

I may use my github.io page occaisionally to write articles to document how to accomplish tasks be they general tech, Linux administration, networking, etc.
They'll all be things that I found insufficient documentation to do.
Hopefully someone else out on the Internet will find my posts useful.

## Links to My Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a hfref="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
