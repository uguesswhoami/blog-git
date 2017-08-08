title: 网络通信之NIO

date: 2016-05-28

categories: [网络通信] 

tags: [网络通信]

---
>  在分布式系统中处处存在网络通信，分布式服务框架、分布式存储系统、分布式缓存都需要高效的网络通信来提高整个系统的运行效率。netty作为最流行的网络通信框架，基于Java NIO模型，被广泛应用于分布式系统中，因此对netty进行系统的学习。
> 特别声明：本节大部分为李林峰的《Netty权威指南》一书的学习笔记。

##  网络通信I/O模型
* 同步阻塞
* 伪异步
* 同步非阻塞
* 异步

以上的网络通信模型特点会按照以下流程介绍：
* 通信模型图
* server-client demo
* 特点分析

<!--more-->
## 同步非阻塞通信模型 -- NIO

* ### 网络模型图

下图为同步非阻塞网络通信模型图。

![NIO网络通信模型.png](http://upload-images.jianshu.io/upload_images/1938652-7ab9df359dca7d37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* ### Server-Client demo

 该demo实现的功能是最简单的c-s网络通信，即客户端请求连接，并向服务器端发出请求，服务器端接收到请求后处理请求并返回结果给客户端。

* Server端

```
public class TimeServer{
    public static void main(String[] args){
        int port = 8888;
        new Thread(new MultiplexerTimeServer(port)).start();
    }
}
```

具体的Server端接收连接请求、处理连接请求代码如下：

```

public class MultiplexerTimeServer implements Runnable{

    private Selector selector;

    private ServerSocketChannel serverSocketChannel;

    private volatile boolean stop = false;

    public MultiplexerTimeServer(int port){

        try {
            //打开多路复用选择器
            selector = Selector.open();
            serverSocketChannel = ServerSocketChannel.open();
            //配置为非阻塞
            serverSocketChannel.configureBlocking(false);
            //backlog设置为1024,未完成握手三次的连接+完成握手三次的连接=1024
            serverSocketChannel.bind(new InetSocketAddress(port), 1024);
            //监听OP_ACCEPT属性,判断是否有连接请求
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        }catch (IOException e){
            e.printStackTrace();
        }
    }
   @Override
    public void run(){
        //选择器开始轮询
        while (!stop){
            try {
                selector.select(1000);
                //返回需要监听的key列表
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                SelectionKey key = null;
                //一个一个开始轮询看是否有处于就绪状态的
                while (it.hasNext()){
                    key = it.next();
                    it.remove();
                    try {
                        //处理处于就绪的状态
                        handleInput(key);
                    }catch (Exception e){
                        if(key != null){
                            key.cancel();
                            if (key.channel() != null){
                                key.channel().close();
                            }
                        }
                    }
                }
            }catch (IOException e){

            }
        }

        if (selector!=null){
            try {
                selector.close();
            }catch (IOException e){
                e.printStackTrace();
            }
        }

    }

    /**
     * 处理就绪的key
     * @param key
     */
    private void handleInput(SelectionKey key) throws IOException{
        if (key.isValid()){
            // 监听到accept事件
            if (key.isAcceptable()){
                //获取就绪的通道
                ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
                SocketChannel sc = ssc.accept();
                //将该通道设置为非阻塞
                sc.configureBlocking(false);
                //监听该通道的读事件
                sc.register(selector, SelectionKey.OP_READ);
            }
            //监听到读事件
            if (key.isReadable()){
                //读取数据
                SocketChannel socketChannel = (SocketChannel)key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                //开始读
                int readByte = socketChannel.read(readBuffer);
                if (readByte > 0){
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String message = new String(bytes, "UTF-8");
                    System.out.println("server recv message: " + message);
                    String currentTime = "QUERY TIME ORDER".equals(message)? new Date(System.currentTimeMillis()).toString(): "BAD";
                    //写结果到客户端
                    doWrite(socketChannel, currentTime);
                }
            }
        }
    }

    private void doWrite(SocketChannel socketChannel, String resp)throws IOException{
        if (resp!=null && resp.trim().length()>0){
            byte[] bytes = resp.getBytes();
            ByteBuffer byteBuffer = ByteBuffer.allocate(bytes.length);
            byteBuffer.put(bytes);
            byteBuffer.flip();
            socketChannel.write(byteBuffer);
        }
    }
}
```
* Client端：

```
public class TimeClient{

    public static void main(String[] args){
        int port = 8888;
        String host = "127.0.0.1";
        new Thread(new MultiplexerTimeClient(host, port)).start();
    }
}
```
具体的Client端请求连接、并接收处理结果的代码如下：

```
public class MultiplexerTimeClient implements Runnable{
    private String host;

    private int port;

    private Selector selector;

    private SocketChannel socketChannel;

    private volatile boolean stop = false;

    public MultiplexerTimeClient(String host, int port){

        this.host = host == null? "127.0.0.1": host;
        this.port = port;

        try {
            //打开多路复用选择器
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            //配置为非阻塞
            socketChannel.configureBlocking(false);

        }catch (IOException e){
            e.printStackTrace();
        }
    }
    @Override
    public void run(){

        try {
            doConnect();
        }catch (IOException e){
            e.printStackTrace();
        }
        //选择器开始轮询
        while (!stop){

            try {
                selector.select();
                //返回需要监听的key列表
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                SelectionKey key = null;
                //一个一个开始轮询看是否有处于就绪状态的
                while (it.hasNext()){
                    key = it.next();
                    it.remove();
                    try {
                        //处理处于就绪的状态
                        handleInput(key);
                    }catch (Exception e){
                        if(key != null){
                            key.cancel();
                            if (key.channel() != null){
                                key.channel().close();
                            }
                        }
                    }
                }
            }catch (IOException e){

            }
        }

        if (selector!=null){
            try {
                selector.close();
            }catch (IOException e){
                e.printStackTrace();
            }
        }

    }

    /**
     * 处理就绪的key
     * @param key
     */
    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            SocketChannel sc = (SocketChannel)key.channel();
            // 监听到connect事件
            if (key.isConnectable()) {
                if (sc.finishConnect()){
                    //监听该通道的读事件
                    sc.register(selector, SelectionKey.OP_READ);
                    doWrite(sc);
                }else {
                    System.exit(1);
                }
            }
            //监听到读事件
            if (key.isReadable()) {
                //读取数据
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                //开始读
                int readByte = socketChannel.read(readBuffer);
                if (readByte > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String message = new String(bytes, "UTF-8");
                    System.out.println("client recv message: " + message);
                    this.stop = true;
                }else if(readByte < 0){
                    key.cancel();
                    sc.close();
                }
            }
        }
    }

    private void doConnect() throws IOException{
       if (socketChannel.connect(new InetSocketAddress(host, port))){
           socketChannel.register(selector, SelectionKey.OP_READ);
           doWrite(socketChannel);
        }else {
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
    }

    private void doWrite(SocketChannel socketChannel)throws IOException{
            byte[] bytes = "QUERY TIME ORDER".getBytes();
            ByteBuffer byteBuffer = ByteBuffer.allocate(bytes.length);
            byteBuffer.put(bytes);
            byteBuffer.flip();
            socketChannel.write(byteBuffer);
            if (!byteBuffer.hasRemaining()){
                System.out.println("send message succeed.");
            }
        }
}
```
* ### 同步非阻塞网络通信模型特点
 * NIO网络通信采用的是I/O多路复用模式，可以将需要监听的通道注册进多路复用选择器中，并设置需要监听该通道的什么事件（可连接or可读or可写），并且将其设置为非阻塞模式，如果没有该事件则继续轮询，当通道有事件时，即做出响应，这样只需一个线程轮询即可，节省了资源的利用。
 * 因为为非阻塞模式，所以某一个客户端的请求不会阻塞掉其他客户端请求，也就不会出现客户端连接超时的情况。