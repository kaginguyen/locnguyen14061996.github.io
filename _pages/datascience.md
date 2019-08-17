---
layout: archive
permalink: /data-science/
title: "Data Science Posts"
author_profile: true

---


{% include group-by-array collection=site.posts field="tags" %}

{% for tag in group_names %}
  {% assign posts = group_items[forloop.index0] %}
  <h2 id="{{ tag | slugify }}" class="archive__subtitle">{{ tag }}</h2>
  {{ posts[0].title }}
  {% for post in posts %}
    {% include archive-single.html %}
  {% endfor %}
{% endfor %}

