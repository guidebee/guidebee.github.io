---
layout: single
title: "Technical References"
permalink: /references/
author_profile: true
---

> **Note:** These are initial design and planning documents created during early development of the Solana Trading System. They reflect the thinking and architecture decisions at the time of writing and **may not be up to date** with the current implementation. Refer to the [blog posts](/year-archive/) for the latest development progress.

## Documentation Index

{% assign sorted_docs = site.docs | sort: "slug" %}
{% for doc in sorted_docs %}
{% assign docname = doc.path | split: "/" | last | split: "." | first %}
- [{{ docname }}]({{ doc.url }})
{% endfor %}
