---
title: archive
excerpt: "All posts"
layout: archive
author_profile: true
permalink: /archive/
---

{% include base_path %}
{% capture written_year %}'None'{% endcapture %}
{% for post in site.posts %}
  {% include archive-single.html %}
{% endfor %}
