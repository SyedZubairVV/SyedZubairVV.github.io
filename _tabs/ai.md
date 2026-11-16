---
title: AI
icon: fas fa-robot
order: 6
---

{% assign section_posts = site.categories['AI'] %}

<ul>
{% for post in section_posts %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <small>({{ post.date | date: "%b %-d, %Y" }})</small>
  </li>
{% endfor %}
</ul>