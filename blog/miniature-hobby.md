---
layout: default
title: Miniature Hobby
parent: Blog
nav_order: 1
---

# Miniature Hobby

Notes and experiments from the hobby desk — painting techniques, brush reviews, and miniature projects.

---

{% assign hobby_posts = site.posts | where_exp: "post", "post.categories contains 'hobby'" | sort: 'date' | reverse %}
{% for post in hobby_posts %}
### [{{ post.title }}]({{ post.url }})
*{{ post.date | date: "%B %d, %Y" }}*

{{ post.excerpt }}

---
{% endfor %}
