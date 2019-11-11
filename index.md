---
title: An Inky Page...
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}{{ post.content }}</a>
    </li>
  {% endfor %}
</ul>

[*Andrew Jorgensen*](http://andrew.jorgensenfamily.us/)
