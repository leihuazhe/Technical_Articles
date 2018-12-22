# Why do we really need multiple netty boss threads?
> 为什么我们需要多个 netty boss线程? 即在 Boss 的 EventLoopGroup 中定义多个线程
  
## 提问者问题
> I'm really confused about the number of threads for a boss group. 
I can't figure out a scenario where we need more than one boss thread.
 In do we need more than a single thread for boss group? 
 the creator of Netty says multiple boss threads are useful if we share NioEventLoopGroup between different server bootstraps,
 but I don't see the reason for it

我对 `BossGroup` 线程池的线程数量非常困惑。我想不出需要多个boss线程的场景。在 `BossGroup` 中是否需要多个线程?
Netty的创建者说，如果我们在不同的 `ServerBootStrap` 之间共享 `NioEventLoopGroup`，那么多个boss线程是有用的，但是我不知道这是为什么?

### Consider this simple Echo server:
考虑如下一个简单的netty服务端例子  EchoServer

```java
public class EchoServer {

    private final int port;
    private List<ChannelFuture> channelFutures = new ArrayList<ChannelFuture>(2);

    public EchoServer(int port) {
        this.port = port;
    }

    public void start() throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup(4);

        for (int i = 0; i != 2; ++i) {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class) // the channel type
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch)
                                throws Exception {
                            System.out.println("Connection accepted by server");
                            ch.pipeline().addLast(
                                    new EchoServerHandler());
                        }
                    });
            // wait till binding to port completes
            ChannelFuture f = b.bind(port + i).sync();
            channelFutures.add(f);
            System.out.println("Echo server started and listen on " + f.channel().localAddress());
        }

        for (ChannelFuture f : channelFutures)
            f.channel().closeFuture().sync();

        // close gracefully
        workerGroup.shutdownGracefully().sync();
        bossGroup.shutdownGracefully().sync();
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println(
                    "Usage: " + EchoServer.class.getSimpleName() +
                            " <port>");
            return;
        }
        int port = Integer.parseInt(args[0]);
        new EchoServer(port).start();
    }
}
```

> In the above example, I create a bossGroup with 1 thread and workerGroup with 4 threads 
and share both event groups to two different bootstraps that bind to two different ports (e.g. 9000 and 9001). 
Below is my handler

在上面的例子中，我创建了一个 带有1个线程的 `bossGroup` 和 带有4个线程的 `workerGroup`，
并将这两个 `EventLoopGroup` 共享给绑定到两个不同端口(例如9000和9001)的两个不同 `BootStrap`,
下面是我的处理程序

```java
@ChannelHandler.Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx,
                            Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Server received: " + in.toString(CharsetUtil.UTF_8) + " from channel " + ctx.channel().hashCode());
        ctx.write(in);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        System.out.println("Read complete for channel " + ctx.channel().hashCode());
        // keep channel busy forever
        while (true) ;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx,
                                Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```
> In my handler above, I purposely keep the channel busy by doing while(true);
Now if I start my app with parameter 9000, 
it will create two server bootstraps that bind at port 9000 and 9001.

在上面的处理程序中，我故意通过 `while(true)` 保持 `channel` 繁忙。
现在，如果我使用绑定端口 `9000` 启动我的应用程序，它将创建绑定在端口9000和9001的两个服务器引导。

```console
Echo server started and listen on /0:0:0:0:0:0:0:0:9090
Echo server started and listen on /0:0:0:0:0:0:0:0:9091
```
> Now, if I connect to both ports and start sending data, 
the maximum # of connections that can be received is 4, which makes sense since I have created 4 worker threads and keep their channel busy without closing it:

 现在，如果我连接到两个端口的程序并开始发送数据, 可以接收到的最大连接数是4，这是有意义的，因为我已经创建了4个工作线程，并保持它们的通道繁忙而不关闭它

```
echo 'abc' > /dev/tcp/localhost/9000
echo 'def' > /dev/tcp/localhost/9000
echo 'ghi' > /dev/tcp/localhost/9001
echo 'jkl' > /dev/tcp/localhost/9000
echo 'mno' > /dev/tcp/localhost/9001 # will not get connected
```

You can also do:

```
telnet localhost 9000 -> then send data "abc"
telnet localhost 9000 -> then send data "def"
telnet localhost 9001 -> then send data "ghi"
telnet localhost 9000 -> then send data "jkl"
telnet localhost 9001 -> # will not get connected
```
> what I don't understand is, I have one boss thread and I'm able to connect to two ports with two server bootstraps. 
So why do we need more than one boss thread (and by default, the # of boss threads is 2*num_logical_processors)?

我不明白的是，我有一个boss线程，我可以用两个 `ServerBootStrap` 连接到两个端口。
那么，为什么我们需要多个boss线程(默认情况下，boss线程的#是2*num_logical_processors)呢?

# Answer

the creator of Netty says multiple boss threads are useful if we share NioEventLoopGroup between different server bootstraps, but I don't see the reason for it

Netty的创建者说，如果我们在不同的服务器引导之间共享NioEventLoopGroup，那么多个boss线程是有用的，但是我不知道这是为什么

> As Norman Maurer said, it's not necessary, but it's very useful.
If you are using 1 thread for 2 different bootstraps, it means that you can't handle connections to this bootstraps simultaneously. So in very very bad case, when boss thread handle only connections for one bootstrap, connections to another will never be handled.

正如 `Norman Maurer` 所说，这没有必要，但非常有用。
如果您为两个不同的引导程序使用一个线程，这意味着您不能同时处理到这个引导程序的连接。
所以在非常非常糟糕的情况下，当boss线程只处理一个引导程序的连接时，将永远不会处理到另一个引导程序的连接。

> Same for workers EventLoopGroup.


> So one boss thread can handle multiple connections, but not simultaneously? And, it is useful to have multiple boss threads (e.g. in a web server where we expect multiple simultaneous connections?) Also, is my understanding or worker threads correct? Thanks

所以一个 `boss` 线程可以处理多个连接，但不能同时处理? 而且，拥有多个 `boss` 线程(例如，在web服务器中，我们希望有多个并发连接?)
另外，我的理解或工作线程是正确的吗?谢谢

boss threads handle connections and pass processing to worker threads. Yeh, it's useful to have multiple boss threads. But no need to much, 'cause in general worker thread works more time than boss thread.

`boss` 线程处理连接并将处理传递给工作线程。是的，拥有多个 `boss` 线程很有用。但不需要太多，因为一般情况下工人线程比老板线程工作的时间要长。

