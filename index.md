---
layout: index
---
{% assign post = site.posts.first %}

<div class="page-header">
  <h1>{{ post.title }} {% if post.tagline %}<small>{{post.tagline}}</small>{% endif %}</h1>
</div>


{{ post.date | date: "%B %d, %Y" }}

{{ post.content }}

{% include JB/comments %}
