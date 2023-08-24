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
  <tr>
    <th>Species</th>
    <th>Isolate</th>
    <th>Type</th>
    <th>Genes</th>
    <th>Accession</th>
  </tr>
  {% for row in site.data.resources %}
    <tr>
      <td>{{ row.Species }}</td>
      <td>{{ row.Isolate }}</td>
      <td>{{ row.Type }}</td>
      <td>{{ row.Genes }}</td>
      <td><a href="{{ row.URL }}">{{ row.Accession }}</a></td>
    </tr>
  {% endfor %}
</table>