---
permalink: /resources/
title: "Resources"
author_profile: true
redirect_from: 
  - /md/
  - /resources.html
---


This is a list of genomes that I have assembled and analyzed:

<table>
  <tr>
    <th>Species</th>
    <th>Isolate</th>
    <th>Type</th>
    <th>Genes</th>
    <th>Accession</th>
    <th>Reference</th>
  </tr>
  {% for row in site.data.resources %}
    <tr>
      <td>{{ row.Species }}</td>
      <td>{{ row.Isolate }}</td>
      <td>{{ row.Type }}</td>
      <td>{{ row.Genes }}</td>
      <td><a href="{{ row.URL }}">{{ row.Accession }}</a></td>
      <td><a href="{{ row.REF }}">{{ row.Reference }}</a></td>
    </tr>
  {% endfor %}
</table>


Scripts and code snippets I used to analyze these genomes are available as repositories in my [GitHub page](https://github.com/alexzaccaron).