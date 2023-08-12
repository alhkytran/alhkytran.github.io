---
layout: default
language: en
---
<ul>
  {% for post in site.posts %}
	{%- if page.language == post.language -%}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
        {%- endif -%}  <!-- if de page.language solo mostrará los del idioma seleccionado -->

  {% endfor %}
</ul>
