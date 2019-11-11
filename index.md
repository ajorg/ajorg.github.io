---
title: An Inky Page...
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.content }}
    </li>
  {% endfor %}
</ul>

[*Andrew Jorgensen*](http://andrew.jorgensenfamily.us/)
