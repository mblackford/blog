---
layout: default
title: Matt Blackford - Technical Blog
---

# 👋 Welcome

I'm Matt, a cloud and DevOps engineer and technical leader. This is where I write about building infrastructure, managing systems at scale, and lessons from the field.

---

## 📝 Recent Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <small>{{ post.date | date: "%B %d, %Y" }}</small><br>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>

---

## 📌 About This Blog

- Text is licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)
- Code is MIT-licensed and free to reuse

[View this blog on GitHub](https://github.com/mblackford/blog)