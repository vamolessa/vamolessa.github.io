# blog posts

{% for post in site.posts %}
- [({{ post.date | date: "%-d %B %Y" }}) {{ post.title }}]({{ post.url }})
{% endfor %}
