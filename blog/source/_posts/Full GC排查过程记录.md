title: Full GC排查过程记录

date: 2017-01-15

categories: [Java] 

---

> 项目中的一次Full GC排查过程

<!--more-->

项目中某一个应用，在客户端访问一直报服务器异常，最终排查是某一个实例不断创建，导致Full GC，这里记录下大体排查过程以及解决办法。

### 问题排查

* 登录服务器查看后台日志时，发现服务器的操作响应极其缓慢，top命令查看CPU使用率和内存占用率都接近100%。因此初步判断有可能是某一个线程一直在死循环并且产生大量对象，占用内存；
* `top -H -p <pid>`查看该Java进程中占用CPU最多的线程为`21002`，转换为十六进制`0x520A`；
* `jstack <pid> > stack.txt`dump 某一进程的所有线程堆栈信息，并查找线程id为`0x520A`的线程，如下：

![thread stack](https://raw.githubusercontent.com/uguesswhoami/pictures/master/thread.png)

> 可以看出是JVM线程一直占用CPU，可以基本确认是在执行GC操作；
* 查看gc.log日志，如下:

![gc](https://raw.githubusercontent.com/uguesswhoami/pictures/master/fullgc.png)

> 发现服务器一直在Full GC，说明有大量长期对象在创建；
* `jmap -histo <pid> > mem.txt`，查看JVM对象创建情况及内存使用情况，如下所示：

![mem use info](https://raw.githubusercontent.com/uguesswhoami/pictures/master/memuse.png)

> 可以看到`com.github.diamond.client.event.EventSource$1`这个对象实例被创建，引起了`FutureTask`、`LinkedBlockingQueue`和`RunnableAdapter`被大量创建。

### 解决问题
* 针对Full GC问题，应急的解决办法是先重启服务器，保证线上应用可用，然后再进一步排查问题所在。重启服务器后，线上已基本可用。
* 上面排查结果已经看出主要原因是`com.github.diamond.client.event.EventSource$1`在被大量创建，这个类是公司用的配置中心的一个类，在用到开关的地方会大量用到该类。而该类又设置的非单例，所以每一个请求访问都会去创建一个实例，

```
	@Override
	public boolean isSingleton() {
		return false;
	}
```
但是创建实例不是很可怕，因为如果为短暂使用的对象会被分配到年轻代使用完成后直接被回收掉了，但是这里的实例并没有被回收掉，为什么？继续排查，发现该类中会创建一个线程：
```
protected void fireEvent(EventType type, String propName, Object propValue) {
		final Iterator<ConfigurationListener> it = listeners.iterator();
		if (it.hasNext()) {
			final ConfigurationEvent event = createEvent(type, propName, propValue);
			while (it.hasNext()) {
				final ConfigurationListener listener = it.next();
				executorService.submit(new Runnable() {
					
					@Override
					public void run() {
						listener.configurationChanged(event);
					}
				});
			}
		}
	}
```
而查看堆栈信息，发现：
```
"config-event-thread-1" #164 prio=5 os_prio=0 tid=0x00007fa01c00d800 nid=0x55b8 waiting for monitor entry [0x00007f9ff452d000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.util.concurrent.CopyOnWriteArrayList$COWIterator.next(CopyOnWriteArrayList.java:1151)
	at com.github.diamond.client.event.EventSource$1.run(EventSource.java:66)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
	- <0x00000000ac8fa0a0> (a java.util.concurrent.ThreadPoolExecutor$Worker)

```
也就说说这个线程一直阻塞在这里，这个对象也就释放不掉。也就是非单例+长期存活对象导致的。
* 将该对象设置为单例的即可解决。