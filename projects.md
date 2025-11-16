---
layout: post-list
title: "Nirhar: Projects"
permalink: "/projects/"
---
# Projects

{% for item in site.data.projects %}
### {{ forloop.index }}. [{{ item.title }}]({{ item.url }})
*{{ item.date}}*

{{ item.summary }}
{% endfor %}