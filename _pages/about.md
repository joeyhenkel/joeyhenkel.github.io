---
layout: collection
title: "About"
permalink: "/about/"
entries_layout: grid
---

{% for page in site.about %}

<a href="{{ page.url }}">
  {{page.title}}</a> - {{page.headline}}
  <img src="{{page.picture}}"><br>
  <hr>


{% endfor %} 

<!-- <ul>
  {% for page in site.about %}
    <li>
      <a href="{{ page.url }}">{{ page.title }}</a>
      - {{ page.headline }}
    </li>
  {% endfor %}
</ul> -->