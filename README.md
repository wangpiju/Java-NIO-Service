# Java-NIO-Service

I want to build a simple restful service about Java NIO and Asynchronous programming.

The flow is below:

web browser ---> ajax --> server (service --> using socket to communicate with one standalone app that take 1 minutes to return)

server --> socket --> app (sleep one minute)

.... async programming....

server <--socket <-- app (when server receiving msg, invoke callback to process)

web browser <-- response (json) <--server

We have three parts: Client, Server, App
 
#Client
```
<!--Client
html: a simple html file contains a button to request ajax call, and each alert the json response data.-->
<!DOCTYPE html>
<html>
    <head>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js"></script>
        <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.2/jquery.min.js"></script>
        <title> test ajax</title>
    <head>
<body>
    <button id="requestAjaxButton" type="button" >Click Me!</button>
</body>
</html> 
<!js: simply, I use Jquery lib to implement ajax call and show the message to the client browser screen.-->
<script>
$("#requestAjaxButton").click(function(){
        $.ajax({
             dataType: "json",
             url: "http://ec2.test.com/reququestAjax?type=simpleString",
             success: function(JData){
                $.each(JData, function() {
                    alert(JData[i].name);
                });
             },
             error: function() {
                   alert("ERROR!!!");
             }
        });
 });
</script>
```

#Server
```
spring mvc: use spring mvc to get the post parameters and then open the webSocket to standalone app 
 @RequestMapping(value = "/requestAjax", method = RequestMethod.POST, produces = "application/json")
public  @ResponseBody  Shop  requestAjax( @RequestParam("type") String type)
  {
 Shop shop = new Shop();
  //validate the input          
  if("simpleString".equals(type)){
```


