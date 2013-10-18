---
layout: default
title: Derek Ludwig
---

{% for post in site.posts %}
  <article>
    {% if forloop.first == false %}
      <hr>
    {% endif %}
    <h3><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h3>
    <h6>{{ post.date | date_to_string }}</h6>
    
    {{ post.excerpt }}
  </article>
  
{% endfor %}

