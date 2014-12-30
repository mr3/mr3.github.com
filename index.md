---
layout: default
---

<div class="container">
    <div class="panel panel-default">
        <div class="panel-heading">
          <h3 class="panel-title"><b>Articles</b></h3>
        </div>
        <div class="list-group">
      	{% for post in site.posts %}
          	<a class="list-group-item" href="{{ post.url }}">{{ post.title }}</a>
      	{% endfor %}
        </div>
    </div>
</div>
