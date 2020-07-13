---
layout: page
title: 越学习，越发现自己的无知
titlebar: algorithm
subtitle: <span class="mega-octicon octicon-repo"></span>  有些技术问题亦是如此，越去深究它，你会发现自己以为很懂的东西，其实自己根本不懂
menu: algorithm
css: ['blog-page.css']
permalink: /algorithm
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='algorithm' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
                        <span class='circle'></span>
                    </div>
                </li>
                {% endif %}
            {% endfor %}
        </ul> 

        <!-- Pagination -->
        {% include pagination.html %}

        <!-- Comments -->
         <div class="comment">
            {% include disqus-comments.html %}
        </div>
         
    </div>
    <script>
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });

    });
</script>

</div>
