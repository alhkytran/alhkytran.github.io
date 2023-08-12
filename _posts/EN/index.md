---
layout: default
language: en
---
<ul>
  {% for post in site.posts %}
	{%- if page.language == post.language -%}
	<div class="divclickable">
		{{ post.title }}
	      <a href="{{ post.url }}"><span class=clickable></span></a>
        {%- endif -%}  <!-- if de page.language solo mostrará los del idioma seleccionado -->

  {% endfor %}
</ul>
