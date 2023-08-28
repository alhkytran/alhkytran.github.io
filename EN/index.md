---
layout: default
language: en
---
<div style="width: 100%;text-align: right;"><a href="javascript:history.back()"><- back</a>&nbsp;&nbsp;|&nbsp;&nbsp;<a href="https://alhkytran.github.io">home</a></div>

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


