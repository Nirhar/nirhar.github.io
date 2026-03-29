---
layout: post-list
title: "Nirhar: Projects"
permalink: "/projects/"
---
# Projects

{{ site.projects_message }}

{% for item in site.data.projects %}
### {{ forloop.index }}. [{{ item.title }}]({{ item.url }})
*{{ item.date}}*

{{ item.summary }}
{% endfor %}