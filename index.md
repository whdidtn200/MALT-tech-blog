---
layout: home
title: MALT Tech Blog
---

# MALT Tech Blog

PHM / Agentic AI / Railway Intelligence 중심 기술 블로그입니다.

- [About](./ABOUT.html)
- [Contact](./CONTACT.html)
- [Privacy Policy](./PRIVACY.html)
- [All Posts](./posts.html)

## 최신 글 아카이브

다음은 `site.posts`를 기준으로 최신 순으로 정렬한 리스트입니다.

<ul>
{% assign sorted_posts = site.posts | sort: "date" | reverse %}
{% for post in sorted_posts limit:8 %}
  <li>
    <small>{{ post.date | date: "%Y-%m-%d" }}</small>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>

> 본 사이트는 GitHub Pages 기준으로 외부 공개 URL에서 접근 가능하도록 구성되었습니다.
