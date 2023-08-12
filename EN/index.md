---
layout: default
language: en
---

  {% for post in site.posts %}
	{%- if page.language == post.language -%}
<ul>
	<div class="divclickable">
		<div class="text-indiv">
			{{ post.title }}
        	    <p>
			{{ post.excerpt | strip_html | truncatewords:75 }}	
	            </p>
		<br>
		<p>tags: {{ post.tags }}</p>
		<p style="text-align: right; padding-right: 30px;">{{ post.date | date: "%-d %B %Y" }}</p>

		      <a href="{{ post.url }}"><span class=clickable></span></a>
         	</div>
	</div>
</ul>
        {%- endif -%}  
  {% endfor %}


