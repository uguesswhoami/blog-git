title: dubbo原理和实践整理

date: 2016-01-01

categories: [dubbo] 

---
> dubbo框架在分布式系统中应用广泛，其核心为服务的RPC远程调用，并在其基础之上实现了集群管理、负载均衡、服务治理、服务监控等功能。

 <!--more-->

### dubbo超时处理
* dubbo 超时源码
```
@Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
            	boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
            	ResponseFuture future = currentClient.request(inv, timeout) ;
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
            	RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
* dubbo重试源码
```
 public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    	List<Invoker<T>> copyinvokers = invokers;
    	checkInvokers(copyinvokers, invocation);
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
        	//重试时，进行重新选择，避免重试时invoker列表已发生变化.
        	//注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
        	if (i > 0) {
        		checkWhetherDestroyed();
        		copyinvokers = list(invocation);
        		//重新检查一下
        		checkInvokers(copyinvokers, invocation);
        	}
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List)invoked);
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + invocation.getMethodName()
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers 
                            + " (" + providers.size() + "/" + copyinvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
                + invocation.getMethodName() + " in the service " + getInterface().getName() 
                + ". Tried " + len + " times of the providers " + providers 
                + " (" + providers.size() + "/" + copyinvokers.size() 
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
    }

}
```
* provider端基本配置：

```
 <dubbo:application name="${application.name}" owner="${dubbo.application.owner}"/>

 <dubbo:protocol name="dubbo" port="${dubbo.service.provider.port}" />

 <dubbo:registry protocol="zookeeper" address="${dubbo.zk.servers}" group="${dubbo.zk.group}" client="zkclient"/>

 <dubbo:monitor address="${dubbo.monitor.address}"/>

 <dubbo:service interface="com.xiaoke.dubbo.DemoService" ref="demoService" loadbalance="random" timeout="3000"/>

```

`当provider端配置超时时间、重试次数后,如果consumer端请求>超时时间就会重试,>重试次数后报异常`

`当provider端未配置超时时间、重试次数或超时时间设置为0时,默认超时时间为1000ms, 默认重试次数为2次,`

` <dubbo:service interface="com.xiaoke.dubbo.DemoService" ref="demoService" loadbalance="random" retries="0"/>`

* dubbo 超时重试机制导致的雪崩

 * 场景
 
   * provider端`timeout`设置为`2000ms`,`retries`设置为`3`;
   * provider端服务处理时间为`3000ms`
   * 此时consumer端调用dubbo服务会造成连接超时进而重试,也就是consumer端共发起4次连接请求,provider端需有四个线程来处理请求,增加了`4`倍，重试机制会不断放大连接请求。

* dubbo超时重试机制总结

  * dubbo默认`timeout=1000ms`,默认`retries=2(不包括本身调用的1次)`,所以要显式配置;
  * 根据provider端的实际业务处理来设置`timeout`;
  * `retries`尽量设置为0,如果需要设置,则应保证provider端业务的幂等性;
  * 如果要配置`retries`,则应防止dubbo服务超时重试导致的服务雪崩。

### dubbo的启动时检查
 > dubbo会在启动时检查服务是否可用,当服务不可用时报异常,默认情况下`check="true"`
 
 * 配置如下:

 ```
 <dubbo:reference interface="com.xiaoke.dubbo.DemoService" id="demoService" check="false"/>
 ```
 做以上配置后,不会做启动时检查,在启动时如果服务还没有启动不会抛出异常,只会在该服务被调用的时候才会抛出异常。
 
 * dubbo启动时检查总结
    * 建议还是使用默认的`check="true"`,这样可以尽早的发现问题
    * 这样在线上发布时就要注意各个服务之间的依赖先后关系,以免在线上发布报异常

### dubbo线程模型配置

* dubbo线程模型原理

* dubbo线程模型配置

```
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

* dispatcher 各种配置 `默认=all` `<以下摘抄自dubbo文档>`
    * all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
    * direct 所有消息都不派发到线程池，全部在IO线程上直接执行。
    * message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在IO线程上执行。
    * execution 只请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在IO线程上执行。
    * connection 在IO线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。
    
* threadpool 各种配置 `默认=fixed` `<以下摘抄自dubbo文档>`
    * fixed 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
    * cached 缓存线程池，空闲一分钟自动删除，需要时重建。
    * limited 可伸缩线程池，但池中的线程数只会增长不会收缩。(为避免收缩时突然来了大流量引起的性能问题)。

* dubbo线程模型配置总结
    * 默认情况下 `threadpool=fixed, threads=200`
    * 如果事件处理的逻辑能迅速完成，并且不会发起新的IO请求，比如只是在内存中记个标识，则直接在IO线程上处理更快，因为减少了线程池调度。
    * 但如果事件处理逻辑较慢，或者需要发起新的IO请求，比如需要查询数据库，则必须派发到线程池，否则IO线程阻塞，将导致不能接收其它请求。
    * 重写线程池(比如要实现新的reject方案或者给线程池加名字或者要使用其他queue),实现`ThreadPool`接口
    * 重写dispatcher方案,实现`Dispatcher`接口


### 直连提供
* 在测试环境需要直连提供 绕过zookeeper
```
-Dcom.xiaoke.xxx.XxxService=dubbo://localhost:20890
```

或者是

```
<dubbo:reference id="xxxService" interface="com.alibaba.xxx.XxxService" url="dubbo://localhost:20890" />
```


### 配置的最佳实践
* 服务端尽可能配置客户端配置
    * 服务端可配置的客户端配置有：超时时间、重试次数、负载均衡算法、并发连接数等
* 服务端尽可能配置服务端的配置
    * 除去基本的配置以外，还可配置线程池大小等
* 本地注册信息缓存配置
    * 可将注册信息缓存到本地文件，这样当zk挂掉后，新启动的客户端也可以通过本地的注册信息找到相应的服务端


### Filter接口

* Filter接口提供invoker被invoke之前或者之后要做的事情,伪代码如下：

```
invoke之前do sth
invoker.invoke()
invoke之后do sth
```
* 举例说明 本地缓存
```
public class CacheFilter implements Filter {

    private CacheFactory cacheFactory;

    public void setCacheFactory(CacheFactory cacheFactory) {
        this.cacheFactory = cacheFactory;
    }

    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (cacheFactory != null && ConfigUtils.isNotEmpty(invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.CACHE_KEY))) {
            Cache cache = cacheFactory.getCache(invoker.getUrl().addParameter(Constants.METHOD_KEY, invocation.getMethodName()));
            if (cache != null) {
                String key = StringUtils.toArgumentString(invocation.getArguments());
                if (cache != null && key != null) {
                    Object value = cache.get(key);
                    if (value != null) {
                        return new RpcResult(value);
                    }
                    Result result = invoker.invoke(invocation);
                    if (! result.hasException()) {
                        cache.put(key, result.getValue());
                    }
                    return result;
                }
            }
        }
        return invoker.invoke(invocation);
    }

}

```

* 在真正refer时会构造Filter链，调用相应服务
```
 private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (filters.size() > 0) {
            for (int i = filters.size() - 1; i >= 0; i --) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    public Result invoke(Invocation invocation) throws RpcException {
                        return filter.invoke(next, invocation);
                    }

                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }
```
 
### 结果缓存
* cacheFilter实现Filter用来缓存热点数据，将数据存在缓存中，在调用远程服务之前先到缓存中取数据
* 配置
```
<dubbo:reference interface="com.xiaoke.dubbo.CacheDemoService" id="cacheDemoService" cache="lru"/>
```
* 默认缓存`size=1000`，当大于`size`时抛弃掉最开始的缓存。
* 对于服务端为幂等性的可以采用结果缓存
* 缓存定制  扩展`AbstractCacheFactory`，比如将Guava cache与dubbo结合设置缓存超时时间


### dubbo服务降级

* 服务端服务降级  本地mock
    * 当出现RPCException时才会执行mock，因此在处理RPCException时可以考虑使用mock，比如网络错误、连接超时等
    *配置 在api包中创建<api name>Mock.java文件，并实现接口，服务端配置如下：


    `
        <dubbo:service interface="com.xiaoke.dubbo.DemoService" ref="demoService" loadbalance="random" timeout="1000" retries="0" mock="true"/>
    `
* 服务降级优化  借助Filter，客户端服务降级通过检查服务异常的次数或者使用开关来判断是否进入服务降级，也可以实现自定义降级方案，并且会尝试正常调用。


### dubbo插件化实现 ExtentionLoader
> dubbo中加载插件的方式是通过SPI协议来实现，dubbo框架已经定义好协议的实现框架和规范，第三方只要按照规范定义自己的插件，dubbo框架就可以加载该插件。

* 具体定义是要在META-INF/dubbo、META-INF/dubbo/internal或者META-INF/services下面以需要实现的接口全面去创建一个文件，名为：`<接口全限定名>`，内容：`配置名=扩展实现类全限定名`假设Filter接口的一个实现类TimeoutFilter，则文夹名称为`com.alibaba.dubbo.rpc.Filter`，内容为：`timeout`