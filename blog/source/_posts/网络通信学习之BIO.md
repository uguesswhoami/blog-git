title: 网络通信之BIO

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

## 同步阻塞网络通信模型 -- BIO

* ### 网络模型图

下图为同步阻塞网络通信模型图。
![同步阻塞通信模型.png](http://upload-images.jianshu.io/upload_images/1938652-58f321f3baa74d2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* ### Server-Client demo

 该demo实现的功能是最简单的c-s网络通信，即客户端请求连接，并向服务器端发出请求，服务器端接收到请求后处理请求并返回结果给客户端。
* Server端
同步阻塞模式的Server端在创建serverSocket后会监听端口，等待接收accept客户端的请求，在等待阶段accept会阻塞，当有客户端请求时，会创建socket并开启处理请求线程；
```
public class TimeServer{

    public static void main(String[] args) throws IOException{
        ServerSocket serverSocket = null;
        int port = 8080;
        try {
            serverSocket = new ServerSocket(port);

            Socket socket = null;
            while (true){
                //循环接收客户端请求
                socket = serverSocket.accept();
                //当接收到请求后,处理请求(接收消息,处理消息)
                new Thread(new TimeServerHandler(socket)).start();
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
实际处理请求代码如下：

```
/**
 * 实际的处理请求类
 */
public class TimeServerHandler implements Runnable{

    private Socket socket;
    public TimeServerHandler(Socket socket){
        this.socket = socket;
    }

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
* ### 同步阻塞网络通信模型特点
由以上的server-client demo可以看出同步阻塞模型有以下特点：
 *  Server端在Client端请求之前一直处于阻塞状态；
 *  当Client端有请求之后会创建新的线程来处理请求并返回请求处理结果；
 
 这样的特点就造成了以下几点弊端：

* 当Client端的请求数量特别多时，这个时候就需要开辟很多个线程来处理请求，线程的创建和销毁都是非常消耗资源的，这样就会造成服务器端内存被撑爆或者是CPU占用率过高等状况，给用户造成的影响就是请求超时。