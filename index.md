---
layout: page
title: 江南·听雨轩
tagline: Coder? Coder!
---
{% include JB/setup %}

{% for post in site.posts %}
<div class = "card">
		<div  class = "date_label">
			<div class="day_month">
      			{{ post.date | date:"%m/%d" }}
      			</div>
      			<div class="year">
      			{{ post.date | date:"%Y" }}
      			</div>
      		</div> 
		{{ post.content  | | split:'<!--break-->' | first }}
	<div class = "read_more">
		<a class="fa fa-link" href="{{ BASE_PATH }}{{ post.url }}">  查看全文&hellip;</a>
	</div>
	
</div>

{% endfor %}

<nav class="pagination" role="navigation">
    {% if paginator.previous_page %}
    <a class="previous pagination__newer btn btn-small btn-tertiary" href="{{ paginator.previous_page_path }}">&larr; 上一页</a>
    {% endif %}
    <span class="page_num pagination__page-number">{{ paginator.page }} / {{ paginator.total_pages }}</span>
    {% if paginator.next_page %}
    <a class="next pagination__older btn btn-small btn-tertiary" href="{{ paginator.next_page_path }}">下一页 &rarr;</a>
    {% endif %}
</nav>

