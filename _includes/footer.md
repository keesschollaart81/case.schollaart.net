<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
<script>window.jQuery || document.write('<script src="/scripts/jquery.min.js"><\/script>')</script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery-backstretch/2.0.4/jquery.backstretch.min.js"></script>
<script>window.jQuery.backstretch || document.write('<script src="/scripts/jquery.backstretch.min.js"><\/script>')</script>
  
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js"></script>
<script>(typeof $().emulateTransitionEnd == 'function') || document.write('<script src="/scripts/bootstrap.min.js"><\/script>')</script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.2.0/highlight.min.js"></script>
<script>window.hljs || document.write('<script src="/scripts/highlight.min.js"><\/script>')</script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/fancybox/2.1.5/jquery.fancybox.min.js"></script>
<script>window.jQuery.fancybox || document.write('<script src="/scripts/jquery.fancybox.min.js"><\/script>')</script>

<script type="text/javascript">
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-58834-13', 'auto');
  ga('send', 'pageview');

  $(document).ready(function () {
		{% if page.backgroundUrl == null %}
			$.backstretch('/img/back1.jpg', { fade: 2000}); 
		{% else %}
			$.backstretch('{{page.backgroundUrl}}', { fade: 2000});
		{% endif %}
		
		$("a.fancybox").fancybox();
    }); 
	
	hljs.configure({
	  tabReplace: '   ' 
	});
	
	hljs.initHighlightingOnLoad(); 
	
	<!-- local css fallbacks -->
	(function($){
		var links = {};

		$( "link[data-fallback]" ).each( function( index, link ) {
			links[link.href] = link;
		});

		$.each( document.styleSheets, function(index, sheet) {
			if(links[sheet.href]) {
				var rules = sheet.rules ? sheet.rules : sheet.cssRules;
				if (rules.length == 0) {
					link = $(links[sheet.href]);
					link.attr( 'href', link.attr("data-fallback") );
				}
			}
		});
	})(jQuery);
</script>