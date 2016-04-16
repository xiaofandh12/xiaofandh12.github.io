---
layout: home
---

<div class="index-content opinion">
    <div class="section">
        <ul class="artical-cate">
            <li><a href="/"><span>技术与阅读</span></a></li>
            <li class="on" style="text-align:center"><a href="/life"><span>生活</span></a></li>
            <li style="text-align:right"><a href="/me"><span>关于我</span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.life %}
            <li>
                <h2>
                    <a href="{{ post.url }}">{{ post.title }}</a>
                </h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
