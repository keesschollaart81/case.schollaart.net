---
layout: layout
--- 
<h1>{{ page.title }}</h1>
<div class="subtitle">{{ page.date | date_to_string }}</div>

{{ content }}
 

  {% if page.comments != null && page.comments == true %}

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
{% endif %}