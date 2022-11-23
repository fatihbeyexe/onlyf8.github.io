---
layout: page
title: DFIR Postları
---

<img title="Under Construction" src="/assets/under-construction.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<ul >
    {% for post in site.posts %}
      {% if post.language == 'TR' and post.categories|lower == 'dfir' -%}
        <li>
            <h2><a href="{{ post.url | prepend: site.baseurl | replace: '//', '/' }}">{{ post.title }}</a></h2>
            <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date_to_string }}</time>
            <p>{{ post.content | strip_html | truncatewords:50 }}</p>
        </li>
      {% endif %}
    {% endfor %}
</ul>