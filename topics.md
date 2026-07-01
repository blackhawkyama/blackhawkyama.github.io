---
layout: page
title: Topics
permalink: /topics/
---

Browse every write-up by category or tag. Built with plain Liquid — no plugins — so it
works on GitHub Pages as-is and updates automatically each time you publish a post.

{% comment %} ---------------- CATEGORIES ---------------- {% endcomment %}
## By category

{% assign sorted_categories = site.categories | sort %}
{% for category in sorted_categories %}
{% assign cat_name = category[0] %}
{% assign cat_posts = category[1] %}
### {{ cat_name }} <span style="font-weight:normal;">({{ cat_posts | size }})</span>
{: id="cat-{{ cat_name | slugify }}"}

{% for post in cat_posts %}
- <span style="white-space:nowrap;">{{ post.date | date: "%b %-d, %Y" }}</span> — [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
{% endfor %}

{% comment %} ---------------- TAGS ---------------- {% endcomment %}
## By tag

{% assign sorted_tags = site.tags | sort %}
{% for tag in sorted_tags %}
{% assign tag_name = tag[0] %}
{% assign tag_posts = tag[1] %}
### {{ tag_name }} <span style="font-weight:normal;">({{ tag_posts | size }})</span>
{: id="tag-{{ tag_name | slugify }}"}

{% for post in tag_posts %}
- <span style="white-space:nowrap;">{{ post.date | date: "%b %-d, %Y" }}</span> — [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
{% endfor %}
