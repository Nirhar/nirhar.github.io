---
layout: post-list
title: Blog Posts
permalink: /blog/
---
# My Blog

{{ site.blog_message }}

{% for post in site.posts %}
<div class="post-list-header">
  <h3>{{ forloop.index }}. <a href="{{ post.url }}">{{ post.title }}</a></h3>
  <i>{{ post.date | date_to_string }}</i>
</div>

{{ post.summary }}
{% endfor %}

