---
permalink: /findings/
title: "Findings"
author_profile: true
redirect_from: 
  - /md/
  - /findings.html
---

These are m
## My major findings of my career as a researcher

<table>
  {% for row in site.data.resources %}
    {% if forloop.first %}
    <tr>
      {% for pair in row %}
        <th>{{ pair[0] }}</th>
      {% endfor %}
    </tr>
    {% endif %}

    {% tablerow pair in row %}
      {{ pair[1] }}
    {% endtablerow %}
  {% endfor %}
</table>