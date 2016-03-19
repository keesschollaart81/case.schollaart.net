<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
{% include head.md %}
<body>
    <div id="container">

		{% include side.md %}
        
        <div id="content">
           
                <div id="homebutton">
                    <a href="/"><span class="glyphicon glyphicon-home"></span></a>
                </div> 
				
				{{ content }} 
        </div>
    </div>
   {% include footer.md %} 
</body>
</html>