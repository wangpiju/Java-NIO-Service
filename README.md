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
#html: a simple html file contains a button to request ajax call, and each alert the json response data.
```
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
#spring mvc: use spring mvc to get the post parameters and then open the webSocket to standalone app 
```
@RequestMapping(value = "/requestAjax", method = RequestMethod.POST, produces = "application/json")
public  @ResponseBody  Shop  requestAjax( @RequestParam("type") String type)
{
 Shop shop = new Shop();
  //validate the input          
  if("simpleString".equals(type)){
       AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();
       SocketAddress serverAddr = new InetSocketAddress("localhost", 8989);
       Future<Void> result = channel.connect(serverAddr);
       
       System.out.println("Connected");
       Attachment attach = new Attachment();
       attach.channel = channel;
       attach.buffer = ByteBuffer.allocate(2048);
       attach.isRead = false;
       attach.mainThread = Thread.currentThread();
       Charset cs = Charset.forName("UTF-8");
       String msg = type;
       byte[] data = msg.getBytes(cs);
       attach.buffer.put(data);
       attach.buffer.flip();
   
       ReadWriteHandler readWriteHandler = new ReadWriteHandler();
       channel.write(attach.buffer, attach, readWriteHandler);
       attach.mainThread.join();
       
       while (!result.isDone()) {
           System.out.println("Task is not completed yet...."); 
           Thread.sleep(1); //sleep for 1 millisecond before checking again 
       } 
       
       System.out.println("Task is completed, let's check result"); 
       shop.setName(future.get());
       return shop;
 }
}

class Attachment {
  AsynchronousSocketChannel channel;
  ByteBuffer buffer;
  Thread mainThread;
  boolean isRead;
}

class ReadWriteHandler implements CompletionHandler<Integer, Attachment> {
  @Override
  public void completed(Integer result, Attachment attach) {
    if (attach.isRead) {
      attach.buffer.flip();
      Charset cs = Charset.forName("UTF-8");
      int limits = attach.buffer.limit();
      byte bytes[] = new byte[limits];
      attach.buffer.get(bytes, 0, limits);
      String msg = new String(bytes, cs);
      System.out.format("Server Responded: "+ msg);
      try {
        msg = this.getTextFromUser();
      } catch (Exception e) {
        e.printStackTrace();
      }
      if (msg.equalsIgnoreCase("bye")) {
        attach.mainThread.interrupt();
        return;
      }
      attach.buffer.clear();
      byte[] data = msg.getBytes(cs);
      attach.buffer.put(data);
      attach.buffer.flip();
      attach.isRead = false; // It is a write
      attach.channel.write(attach.buffer, attach, this);
    }else {
      attach.isRead = true;
      attach.buffer.clear();
      attach.channel.read(attach.buffer, attach, this);
    }
  }
  @Override
  public void failed(Throwable e, Attachment attach) {
    e.printStackTrace();
  }
}
       
```

#App
#create an NIO server to listen the request 
```
public static void main(String[] args) throws Exception {
 AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel
        .open();// www  .  ja va2 s  .c om
    String host = "localhost";
    int port = 8989;
    InetSocketAddress sAddr = new InetSocketAddress(host, port);
    server.bind(sAddr);
    System.out.format("Server is listening at %s%n", sAddr);
    Attachment attach = new Attachment();
    attach.server = server;
    server.accept(attach, new ConnectionHandler());
    Thread.currentThread().join();
}
    
class Attachment {
  AsynchronousServerSocketChannel server;
  AsynchronousSocketChannel client;
  ByteBuffer buffer;
  SocketAddress clientAddr;
  boolean isRead;
}

class ConnectionHandler implements
    CompletionHandler<AsynchronousSocketChannel, Attachment> {
  @Override
  public void completed(AsynchronousSocketChannel client, Attachment attach) {
    try {
      SocketAddress clientAddr = client.getRemoteAddress();
      System.out.format("Accepted a  connection from  %s%n", clientAddr);
      attach.server.accept(attach, this);
      ReadWriteHandler rwHandler = new ReadWriteHandler();
      Attachment newAttach = new Attachment();
      newAttach.server = attach.server;
      newAttach.client = client;
      newAttach.buffer = ByteBuffer.allocate(2048);
      newAttach.isRead = true;
      newAttach.clientAddr = clientAddr;
      client.read(newAttach.buffer, newAttach, rwHandler);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  @Override
  public void failed(Throwable e, Attachment attach) {
    System.out.println("Failed to accept a  connection.");
    e.printStackTrace();
  }
}

class ReadWriteHandler implements CompletionHandler<Integer, Attachment> {
  @Override
  public void completed(Integer result, Attachment attach) {
    if (result == -1) {
      try {
        attach.client.close();
        System.out.format("Stopped   listening to the   client %s%n",
            attach.clientAddr);
      } catch (IOException ex) {
        ex.printStackTrace();
      }
      return;
    }
    
    if (attach.isRead) {
      attach.buffer.flip();
      int limits = attach.buffer.limit();
      byte bytes[] = new byte[limits];
      attach.buffer.get(bytes, 0, limits);
      Charset cs = Charset.forName("UTF-8");
      String msg = new String(bytes, cs);
      System.out.format("Client at  %s  says: %s%n", attach.clientAddr,
          msg);
      attach.isRead = false; // It is a write
      attach.buffer.rewind();
    } else {
      //sleep 1 min
      Thread.sleep( 60000 );
      // Write to the client
      attach.client.write(attach.buffer, attach, this);
      attach.isRead = true;
      attach.buffer.clear();
      attach.client.read(attach.buffer, attach, this);
    }
  }

  @Override
  public void failed(Throwable e, Attachment attach) {
    e.printStackTrace();
  }

```


