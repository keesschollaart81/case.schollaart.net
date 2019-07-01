---
layout: layout
--- 
           
<div id="homebutton">
    <a href="/"><span class="glyphicon glyphicon-chevron-left"></span></a>
</div> 

<h1>{{ page.title }}</h1>

<br/>
<div class="subtitle">
{{ page.date | date_to_string }}
<span>/</span>
Kees Schollaart
<span>/</span>
<a href="https://nl.linkedin.com/in/keesschollaart" target="_blank"><i class="fa fa-linkedin-square"></i></a>
<a href="https://github.com/keesschollaart81" target="_blank"><i class="fa fa-github-square"></i></a>
<a href="https://twitter.com/keesschollaart" target="_blank"><i class="fa fa-twitter-square"></i></a>
<a href="http://www.emailmeform.com/builder/form/6cNG0B3bIfEp232ftoKR2zO7" target="_blank"><i class="fa fa-envelope-square"></i></a>
</div>

<div id="postBody">
{{ content }}
  
<div id="disqus_thread"></div>
<script>

    var disqus_developer = 1;

    var disqus_config = function () { 
        this.page.identifier = '{{page.title}}';
    };

    (function () { // DON'T EDIT BELOW THIS LINE
        var d = document, s = d.createElement('script');

        s.src = '//caseschollaartnet.disqus.com/embed.js';

        s.setAttribute('data-timestamp', +new Date());
        (d.head || d.body).appendChild(s);
    })();
</script> 
</div>