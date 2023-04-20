# blog posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

{% for post in site.posts %}
- [page.date{{ post.title }}]({{ post.url }})
{% endfor %}
