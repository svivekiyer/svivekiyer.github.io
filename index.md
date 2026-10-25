---
layout: home
title: Vivek Iyer
---

Infrastructure engineer — storage, SAN, backup.

I write about systems I have built, and about the gaps I keep finding in them.

{% assign recent = site.articles | sort: "date" | reverse %}
{% if recent.size > 0 %}

## Writing

<ul class="section-list">
{% for doc in recent limit: 4 %}
  <li>
    <a href="{{ doc.url | relative_url }}">{{ doc.title }}</a>
    {% if doc.summary %}<br><span>{{ doc.summary }}</span>{% endif %}
  </li>
{% endfor %}
</ul>

{% endif %}

[GitHub](https://github.com/svivekiyer) · [LinkedIn]({{ site.linkedin_url }})
