---
layout: page
title: Just Write Something
tagline: Fighting
---
{% include JB/setup %}

<div class="posts">
  {% for post in site.posts %}
    <div class="post-item">
      <div><a class="post-title" href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></div>
      <div class="post-time"><i>Posted in</i> &raquo; <span>{{ post.date | date_to_string }}</span></div>
      <div class="post-excerpt"><blockquote>{{ post.excerpt }}</blockquote></div>
    </div>
  {% endfor %}
</div>
