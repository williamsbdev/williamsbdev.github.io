---
layout: page
---
{% include JB/setup %}

{% assign post = site.posts.first %} {% assign content = post.content %}

<h2>{{ post.title }}</h2>

<h5>{{ post.tagline }}</h5>

{{ post.date | date: "%B %d, %Y" }}

{{ content }}
