---
layout: default
title: 首页
---

# 📚 我的个人知识总结

欢迎来到我的技术笔记空间！这里记录了我的学习心得和技术总结。

## 最新文章

<ul class="post-list">
  {% for post in site.posts %}
    <li class="post-item">
      <h2>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h2>
      <p class="post-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">
          {{ post.date | date: "%Y年%m月%d日" }}
        </time>
        {% if post.categories %}
        | 分类: {{ post.categories | join: ", " }}
        {% endif %}
      </p>
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 150 }}</p>
    </li>
  {% endfor %}
</ul>
