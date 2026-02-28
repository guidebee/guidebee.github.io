---
layout: single
title: "Technical References"
permalink: /references/
author_profile: true
---

Comprehensive technical documentation for the Solana Trading System, covering architecture, implementation plans, design decisions, and operational guides.

## Documentation Index

{% assign sorted_docs = site.docs | sort: "slug" %}
{% for doc in sorted_docs %}
- [{{ doc.title }}]({{ doc.url }})
{% endfor %}
