---
title: "Writeups"
permalink: /writeups/
layout: single
author_profile: true
toc: true
toc_sticky: true
---

External articles and writeups I've published.

{% for entry in site.data.writeups %}
## [{{ entry.title }}]({{ entry.link }})

{{ entry.description }}

*Published on {{ entry.source }}*

---
{% endfor %}
