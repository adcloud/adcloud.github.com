---
layout: page
title: Adcloud Dev!
tagline: Dev insights from Adcloud
---
{% include JB/setup %}

This is the development blog from Adcloud. Where we want to write all things related to development at Adcloud. Be sure to stop by our [team page](http://adkla.us).

## Blog Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></br>{{ post.description }}</li>
  {% endfor %}
</ul>


