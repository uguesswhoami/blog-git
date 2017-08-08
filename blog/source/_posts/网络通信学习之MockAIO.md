title: 网络通信之MockAIO

date: 2016-05-27

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

## 伪异步网络通信模型 -- MockAIO

* ### 网络模型图

下图为伪异步网络通信模型图。

![伪异步通信模型.png](http://upload-images.jianshu.io/upload_images/1938652-4b33745f2500a45b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* ### Server-Client demo

 该demo实现的功能是最简单的c-s网络通信，即客户端请求连接，并向服务器端发出请求，服务器端接收到请求后处理请求并返回结果给客户端。

* Server端

```
public class TimeServer{
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = null;
        int port = 8080;
        try {
            serverSocket = new ServerSocket(port);
            Socket socket = null;
            while (true){
                //循环接收客户端请求
                socket = serverSocket.accept();
                //当接收到请求后,处理请求(接收消息,处理消息) 将处理过程包装为task扔到线程池中
                ThreadExecutorsUtil.execute(new TimeServerHandlerTask(socket));
            }
        }finally {
            if (serverSocket != null){
                serverSocket.close();
                serverSocket = null;
            }
        }
    }
}

```

线程池代码如下：
```
public class ThreadExecutorsUtil{
    private static Executor executor = new ThreadPoolExecutorExtention(20);

    public static void execute(Runnable runnable){
        executor.execute(runnable);
    }
}
```
其中ThreadPoolExecutorExtention是对原ThreadPoolExecutor的扩展，代码如下：
```
public class ThreadPoolExecutorExtention extends ThreadPoolExecutor{
    private long time;

   public ThreadPoolExecutorExtention(int nThreads){
        super(nThreads, nThreads,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());
    }
    @Override
    public void beforeExecute(Thread t, Runnable r){
        System.out.println("task start...");
        super.beforeExecute(t, r);
        time = System.currentTimeMillis();
    }

    @Override
    public void afterExecute(Runnable r, Throwable t) {
        System.out.println("task end... time is: " + (System.currentTimeMillis() - time));
        super.afterExecute(r, t);
    }
}
```

来自客户端的请求的处理会被封装为Task，具体Task如下：

```
public class TimeServerHandlerTask implements Runnable{

    private  Socket socket;
    public TimeServerHandlerTask(Socket socket){
        this.socket = socket;
    }
    @Override
    public void run(){
        //创建输入流、输出流
        ObjectInputStream in = null;
        ObjectOutputStream out = null;
        try {
            //获取输入流输出流
            in = new ObjectInputStream(this.socket.getInputStream());
            out = new ObjectOutputStream(this.socket.getOutputStream());

            //开始读取消息
            String message = null;
            String currentTime = null;
            while (true){
                Object obj = in.readObject();
                if (obj instanceof Throwable){
                    System.out.println("异常。。。");
                }
                message = obj == null? null : obj.toString();
                if (message == null) break;
                //处理消息
                System.out.println("time server recv message: " + message);
                currentTime = "QUERY TIME ORDER".equals(message)? new Date(System.currentTimeMillis()).toString(): "BAD";
                //处理后消息发给客户端
                out.writeObject(currentTime);
            }
            try {
                Thread.sleep(100);
            }catch (InterruptedException e){

            }
        }catch (Exception e){
            e.printStackTrace();
        }finally{
            //关闭流
            if (in != null){
                try {
                    in.close();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }

            if (out != null){
                try {
                    out.close();
                    out = null;
                }catch (Exception e){
                    e.printStackTrace();
                }
            }

            if(this.socket != null){
                try {
                    this.socket.close();
                }catch (Exception e){
                    e.printStackTrace();
                }
                this.socket = null;
            }
        }
    }
}
```

 * Client端

```
public class TimeClient{
    public static void main(String[] args){
        int port = 8080;
        Socket socket = null;
        ObjectOutputStream out = null;
        ObjectInputStream in = null;
        try {
            socket = new Socket("127.0.0.1", port);
            out = new ObjectOutputStream(socket.getOutputStream());
            out.writeObject("QUERY TIME ORDER");
            out.writeObject(null);
            in = new ObjectInputStream(socket.getInputStream());
            String resp = in.readObject().toString();
            System.out.println("recv msg: " + resp);
        }catch (Exception e){

        }finally {
            //关闭流
            if (out != null){
                try {
                    out.close();
                    out = null;
                }catch (Exception e){
                    e.printStackTrace();
                }
            }

            if (in != null){
                try {
                    in.close();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }

            if(socket != null){
                try {
                    socket.close();
                }catch (Exception e){
                    e.printStackTrace();
                }
                socket = null;
            }
        }
    }
}
```

* ### 伪异步网络通信模型特点

由以上的server-client demo可以看出伪异步模型有以下特点：
  *  伪异步与同步阻塞模型最大的不同就是将来自Client端的请求处理包装成Task然后扔进线程池中，由线程池中的线程来执行Task，这样保证了线程数量的固定，不用每一个请求开一个线程，避免资源的浪费；
* 线程池的大小和等待队列的大小都是可控的，可以根据请求数量来设置，达到资源利用的最大化；
 
 这样的特点会造成以下几点弊端：
因为在Task中设计到流的读取操作，如下所示：
```
 Object obj = in.readObject();
```
这个操作是阻塞的，也就是当客户端没有发数据时，线程池中的该线程会一直阻塞在这，假设一种极端的情况：如果所有请求的客户端都没有写数据，也就是线程池中的所有线程全部阻塞在读操作上，那么当有其他客户端接入请求时，会被扔到等待队列中，如果客户端请求的数量继续增大，大到超过了等待队列的容量，这个时候服务器端就已经饱和，无法再处理其他请求，造成客户端的连接超时。

也就是说，其实这种模式看起来像是异步，其实还是阻塞的，所以叫其伪异步通信模式。
