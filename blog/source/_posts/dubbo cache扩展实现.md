title: dubbo cache 扩展

date: 2016-05-01

categories: [dubbo] 

---
> dubbo框架在分布式系统中应用广泛，其核心为服务的RPC远程调用，并在其基础之上实现了集群管理、负载均衡、服务治理、服务监控等功能。本文对dubbo原有缓存进行扩展以达到缓存可配置的目的。

 <!--more-->
 
 ## 场景
 *  目前使用比较多的应用场景是：通过dubbo获取远程数据，并缓存到本地，设置相应的缓存过期时间，在过期时间之内直接从缓存中获取，超过缓存过期时间则由dubbo从远程获取。 伪代码如下：
 
 ```
  set(){
    sth = getSth(); //通过dubbo获取远程数据
    cache.setExpire(time); //设置缓存过期时间
    cache.putIfAbsent(sth); //将数据放入缓存
    }
    
  get(){
    if(sth=cache.get()!=null){         //如果缓存中有数据则直接返回缓存中数据
        return sth;
    }else{
        sth = getSth(); //继续通过dubbo从远程获取
        return sth；
    }
  }
```


## 问题
* 这样做的问题是在每次的dubbo请求中都需要做这样的缓存操作，造成缓存设置散落在各个地方，不容易维护，同时造成冗余代码。

## 实现目标
* 通过扩展dubbo原有的缓存策略，实现缓存超时时间`expireTime`以及缓存大小`size`可配置，这样只要应用方通过配置即可将远程获取的数据转存入缓存并且由应用方决定缓存配置

## 具体实现
> 在平常的业务用应用比较多的缓存有Guava和redis等，本次扩展针对Guava缓存，redis扩展基本相类似。

> 具体的实现分为以下几步：

* 对原有dubbo缓存扩展只要实现dubbo预留出的扩展接口`AbstractCacheFactory`以及`Cache`。其中`AbstractCacheFactory`接口预留出`createCache`接口，用来创建缓存对象；`Cache`接口预留出`put`和`get`接口，用来对数据进行缓存的写和读操作
* 对缓存的配置实现。使用`<file>.properties`文件来配置缓存的`expire time`和`size`，通过读取`.properties`文件将配置加载入缓存，其中配置文件读取的优先级为：`dubbo.properties`中`cache.properties.file`属性指定的文件>`dubbo.properties`的配置>`classpath`下的`application.properties`文件>`classpath:/META-INF`下的`application.properties`文件；其中配置的优先级为：方法级>接口级>缓存公用默认级>默认级
* 具体的代码可参考[这里](https://github.com/uguesswhoami/dubbo-util/tree/master/dubbo-cache)
