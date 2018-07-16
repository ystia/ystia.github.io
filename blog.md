---
title: Ystia suite
layout: default
description: Develop and deploy applications over hybrid infrastructures (IaaS, HPC schedulers, CaaS).
org_count: 60
permalink: /blog/
---
<div class="home">

  <h2 class="alt-h2">Posts</h2>

  <ul class="list-style-none">
    {% for post in site.posts %}
      <li>
        <a class="alt-h3" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        <span class="float-right ml-2">{{ post.date | date: "%b %d, %Y" }}</span>
        {% if post.tags.first == 'index' %}
          {% for tag in post.tags offset:1 %}
            <span class="float-right Label bg-ystia mr-1">{{ tag }}</span>
          {% endfor %}
        {% endif %}
      </li>
    {% endfor %}
  </ul>

</div>