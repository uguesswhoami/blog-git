title: RPC原理简析

date: 2016-05-23 

categories: RPC

tags: [rpc]

---
> 本文主要介绍RPC的原理并实现一个简单的例子。

# RPC主要原理
*  什么是RPC
RPC -- Remote Procedure Call Protocol,是远程过程调用协议的简称。
* 为什么用RPC
在项目开发过程中，接口或者方法的调用分为本地调用和远程调用两种：
本地调用：本地调用很简单，即普通的在一个虚拟机内或者说一个进程内的方法或者接口之间的调用；
远程调用：在一个系统中会存在多个服务且部署在不同的机器上，当A服务器应用需要调用B服务器的应用，就需要用到远程调用，让A像调用本地接口一样调用B服务器中的接口，如图所示。

<!--more-->

![RPC通信图.png](http://upload-images.jianshu.io/upload_images/1938652-0999f48136c83c44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在远程调用中需要遵循一种协议，即远程过程调用协议。
* 实现RPC需要解决的问题
从图中的流程来一一分解RPC需要解决的问题：
(1)网络传输：在底层需要进行数据的网络传输，包括需要调用接口的名称、参数、参数类型等，所以高效的网络传输是需要的；
(2)序列化：从图中可以看出服务器A需要将参数(包括接口名称、参数、参数类型等)传输给B服务器，在服务器接收到参数并调用相应的接口后生成返回值，并且需要将返回值通过网络回传给服务器A，而在网络中传输的是二进制，因此要将需要传输的值序列化为二进制形式，所以高效好用的序列化框架也是需要的。

# RPC简单实现
本节将给出一个简单的RPC实现，网络传输是通过裸的socket编程，序列化是用的原始的序列化方式完成。高性能的网络传输方式和序列化方式在其他文章中整理。
* 服务器端
服务器端主要实现的功能是暴露服务，即监听客户端的调用服务请求并根据客户端的请求参数完成服务所要求的功能，并可以返回给结果给客户端。代码如下：
`/** * 暴露服务
 *@param service 需要暴露的服务 
 *@param port 端口名称
 */`
```java
    public static void export(final Object service, int port) throws Exception{
        if (service == null){
            throw new IllegalArgumentException("service is null");
        }
        if (port < 0 || port > 65535){
            throw new IllegalArgumentException("port is invalid, port is:" + port);
        }
        System.out.println("export service :" + service.getClass().getName() + "on port:" + port);

        //创建socket
        ServerSocket serverSocket = new ServerSocket(port);
        for (;;){
            //1.接收socket连接，当没有连接时会阻塞
            Socket socket = serverSocket.accept();
            //2.当有连接时，扔给线程池
            ExecutorUtils.execute(new ServerTask(socket, service));
        }
    }
```

处理代码如下：
```
public class ServerTask implements Runnable{
    private final Socket socket;
    private final Object service;

    public ServerTask(Socket socket, Object service){
        this.socket = socket;
        this.service = service;
    }
    @Override
    public void run(){
        try {
            //处理连接,等待IO有数据
            ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
            //当有数据时,获取服务方法名称、方法参数、方法参数类型
            try {
                String methodName = objectInputStream.readUTF();
                Class<?>[] parameterTypes = (Class<?>[])(objectInputStream.readObject());
                Object[] arguments = (Object[])objectInputStream.readObject();

                ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                try {
                    Method method = service.getClass().getMethod(methodName, parameterTypes);
                    Object result = method.invoke(service,arguments);
                    objectOutputStream.writeObject(result);
                }catch (Throwable e){
                    e.printStackTrace();
                }finally {
                    objectOutputStream.close();
                }
            }catch (ClassNotFoundException e){
                e.printStackTrace();
            }finally {
                objectInputStream.close();
            }
        }catch (IOException e){

        }finally {
            try {
                socket.close();
            }catch (IOException e){
                e.printStackTrace();
            }
        }
    }
}
```
* 客户端
客户端的功能就是可以向服务器发出请求，并可以将接口参数发送给服务器端，并等待接口服务器返回的执行结果。

```java
 public static <T> T refer(Class<T> interfaceClass, final String host, final int port ){
        if (interfaceClass == null){
            throw new IllegalArgumentException("interfaceClass is null");
        }
        if (StringUtils.isEmpty(host)){
            throw new IllegalArgumentException("host is null");
        }
        if (port < 0 || port > 65535){
            throw new IllegalArgumentException("port is invalid, port is:" + port);
        }

        return (T)Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[]{interfaceClass}, new RpcInvoker(host, port));
    }
```

```java
public class RpcInvoker implements InvocationHandler{

    private String host;

    private int port;

    public RpcInvoker(String host, int port){
        this.host = host;
        this.port = port;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable{
        Socket socket = new Socket(host, port);
        try {
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            objectOutputStream.writeUTF(method.getName());
            objectOutputStream.writeObject(method.getParameterTypes());
            objectOutputStream.writeObject(args);
            //等待服务端返回数据
            ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
            try {
                Object result = objectInputStream.readObject();
                if (result instanceof Throwable){
                    throw (Throwable) result;
                }
                return result;
            }catch (Throwable e){
                e.printStackTrace();
            }finally {
                objectInputStream.close();
            }
        }catch (Throwable e){
            e.printStackTrace();
        }finally {
            socket.close();
        }
        return null;
    }
}
```