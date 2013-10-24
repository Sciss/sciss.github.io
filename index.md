---
layout: page
title: index
tagline: .html
---
{% include JB/setup %}

For the GitHub software projects, visit [github.com/Sciss](https://github.com/Sciss).

&nbsp;

## Recent Blog Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

