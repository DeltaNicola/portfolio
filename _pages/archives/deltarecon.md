---
title: deltarecon's archive
layout: archive
permalink: /posts/deltarecon/
project: deltarecon
entries_layout: list
---
{% assign filtered_posts = site.posts | where_exp: "post", "post.tags contains page.project" %}

<div class="entries-list">
{% for post in filtered_posts %}
  {% include archive-single.html %}
{% endfor %}
</div>

{% if filtered_posts.size == 0 %}
<p>Nessun post trovato per questo progetto.</p>
{% endif %}