---
layout: page
title: DFIR Blog Posts
---


<img title="Under Construction" src="/assets/under-construction.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<ul >
    {% for post in site.posts %}
      {% if post.categories|lower == 'dfir' -%}
        <li>
            <h2><a href="{{ post.url | prepend: site.baseurl | replace: '//', '/' }}">{{ post.categories|lower }}</a></h2>
            <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date_to_string }}</time>
            <p>{{ post.content | strip_html | truncatewords:50 }}</p>
        </li>
      {% endif %}
    {% endfor %}
</ul>
