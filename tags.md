---
layout: default
title: "Tags"
description: "tags"
---

{% for tag in site.tags %}
    <div class="blog-post">
        <h2 class="blog-post-title">
            <a name="{{ tag[0] }}">{{ tag[0] }}</a>
        </h2>
        <p>
          <ul>
            {% for post in tag[1] %}
            <li>
            <span>{{ post.date | date:"%Y-%m-%d" }}</span>
            <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a>
            </li>
            {% endfor %}
          </ul>
        </p>
    </div>
{% endfor %}