---
layout: single
title: posts
permalink: /posts/
---

a collection of posts about automation, ai, devops and some random experiments  
project releases, fixes that made me curse, and everything I learn along the way  

nothing too serious, just sharing experiences and ideas for those who love tinkering with technology

***
{% for post in site.posts %}
  <article>
    <h3>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </h3>
    <time datetime="{{ post.date | date: "%Y-%m-%d" }}"><small>{{ post.date | date_to_long_string }}</small></time>
    <p><small>{{ post.excerpt }}</small></p>
  </article>
{% endfor %}