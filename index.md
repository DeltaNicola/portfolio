---
layout: home
author_profile: true
---

## building stable, secure and scalable systems

every project starts with a devops approach that puts automation, observability and resilience at the center

the goal is to create solutions that are not only functional, but also sustainable and easy to maintain over time, where infrastructure becomes an integral part of the product

#### projects
---

{% capture written_label %}'None'{% endcapture %}

{% for collection in site.collections %}
  {% for post in collection.docs %}
    {% unless collection.output == false or collection.label == "posts" %}
      {% include archive-single.html %}
    {% endunless %}
  {% endfor %}
{% endfor %}

<br />