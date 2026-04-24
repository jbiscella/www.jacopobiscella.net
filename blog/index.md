---
layout: default
title: Blog
nav_order: 7
has_children: true
---

# Blog

Thoughts on engineering, leadership, and hobbies.

---

{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts %}
### [{{ post.title }}]({{ post.url }})
*{{ post.date | date: "%B %d, %Y" }}*

{{ post.excerpt }}

---
{% endfor %}
