---
layout: default
pagination:
  data: collections
  size: 1
  alias: tag
permalink: /tags/{{ tag }}/
eleventyComputed:
  title: "{{ tag }}"
---  

{% for post in collections[tag] %}
<div class="py-4 sm:py-10">
  <p>
    <span class="text-2xl sm:text-4xl font-bold hover:underline"><a href="{{ post.url }}">{{ post.data.title }}</a></span>
  </p>
  <em>{{ post.date | postDate }}</em>
  <p class="mt-4">{{ post.data.post_excerpt | md | safe }}
  <p class="">{% if post.data.thumbnail %}{{ post.data.thumbnail | safe }}{% endif %}</p>
   <a class="flex-none hover:underline font-semibold text-indigo-400" href="{{ post.url }}">Read more &rarr;</a>
  </p>
</div>
{% endfor %}

