---
layout: page
titles:
  en: Categories
  zh-CN: 分类
  zh-Hans: 分类
  zh: 分类
permalink: /categories.html
---

{%- for category in site.categories -%}
{%- assign _posts = category[1] | sort: 'date' | reverse -%}

## {{ category[0] }} <small>({{ _posts | size }} 篇)</small>

{%- for post in _posts -%}

- <a href="{{ post.url | relative_url }}">{{ post.title }}</a> <span class="text-secondary">({{ post.date | date: "%Y-%m-%d" }})</span>

{%- endfor -%}
{%- endfor -%}
