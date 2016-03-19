---
layout : layout
title : case.schollaart.net
Homebutton: false
--- 

<h1>Blog of Kees Schollaart</h1>
<div class="subtitle">http://case.schollaart.net/</div>
<p>
    Hi there! I'd love to code C# and thats what I blog about!<br /><br />

    Recent posts:
</p>

<ul class="posts">
	{% for post in site.posts %}
      <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
	
    <!--@foreach (var post in Model.Site.Posts)
    {
        <li>
             <a href="@post.Url">@post.Title</a> <span class="postdate">@post.Date.ToString("d MMM, yyyy")</span>
        </li>
    }-->
</ul>
