---
layout: page
title: Categories
permalink: /categories/
---

{% for category in site.categories %}
  <h3 id="{{ category[0] }}">{{ category[0] }}</h3>
  <ul>
    {% assign pages_list = category[1] %}
    {% for post in pages_list %}
        {% if post.title != null %}
            {% if group == null or group == post.group %}
            <li>
                <a href="{{ site.url }}{{ post.url }}">{{ post.title }}
                    <time datetime="{{ post.date | date_to_xmlschema }}" itemprop="datePublished">{{ post.date | date: "%B %d, %Y" }}</time>
                </a>
            </li>
            {% endif %}
        {% endif %}
    {% endfor %}
    {% assign pages_list = nil %}
    {% assign group = nil %}
  </ul>
{% endfor %}
