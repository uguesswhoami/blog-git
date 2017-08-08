title: Guava cache 源码浅析

date: 2016-03-01

categories: [Guava] 

---


> 在实际项目中会应用Guava Cache做本地缓存，这里简单分析一下Guava Cache的原理。

<!--more-->

> 在开发过程中要将数据保存到本地内存中，可以使用ConcurrentHashMap，但是要使本地内存有一些新的特性，比如：超时设置、动态加载等，则可以使用Guava cache。
Guava cahce具有以下常用特征：
* 读、写、移除等接口
* 动态加载功能
* 基于不同算法的回收策略
* 缓存击中等性能统计

## Guava cache 数据结构

在ConcurrentHashMap出现之前，JDK早期版本中实现了线程安全的HashMap--HashTable，但是HashTable的实现方式较重（锁住整个表）因此被淘汰；而ConcurrentHashMap使用了`分段锁`技术并发性能更好，Guava cache也借鉴了这种并发处理方式。如图所示。

![0_1478862208560_guava数据结构.png](/uploads/files/1478862208747-guava数据结构-resized.png) 

 > Guava cache由segment数组和Entry链数组组成，cache中Segments定义如下：
 
```
final LocalCache.Segment<K, V>[] segments;
```
Segments中table定义如下：

```
static class Segment<K, V> extends ReentrantLock {
        ...
    volatile AtomicReferenceArray<LocalCache.ReferenceEntry<K, V>> table;
        ...
        }
```
可以看到：
* segment类继承`ReentrantLock`类，说明每个`Segment`拥有一把锁并可以显式的加锁（lock()）、解锁（unlock()），这样当线程1在更新Segment[1]的数据时，其获取到的是Segment[1]的锁，其他线程在更新Segment[1]的数据时会阻塞等待，但是并不会阻塞其他线程更新Segment[i] (i != 1)的数据。
* table是一个存储`ReferenceEntry`类型的原子引用数组，而`ReferenceEntry`类实例中存储的才是真正的key-value对。


## segment定位
由`Guava cache`的数据结构可以看出，要对cache做`put`、`get`等操作首先需定位到相应的`Segment`段，然后定位到具体的table，然后定位到具体的`ReferenceEntry`，进而根据`key`来`get`或者`put`值。于是就出现这样的问题：如何在保证高并发性能的前提下更快的定位到数据?如果将所有的table放在一个`segment`里，这样存取不仅缓慢而且当多线程访问时就会将整个cache锁住，也就失去了分段锁的意义，并发性能槽糕，因此需将不同的entry链均匀的分布在不同的`Segment`中，为达到这样的目的，在将`key`取`hashcode`的基础之上进行一次再`hash`，如下所示：

```
static int rehash(int h) {
        h += h << 15 ^ -12931;
        h ^= h >>> 10;
        h += h << 3;
        h ^= h >>> 6;
        h += (h << 2) + (h << 14);
        return h ^ h >>> 16;
    }
```
然后通过以下算法定位到具体的Segment：
```
    LocalCache.Segment<K, V> segmentFor(int hash) {
        return this.segments[hash >>> this.segmentShift & this.segmentMask];
    }
```
其中`segmentShift`表示segment的偏移量，`segmentMask`表示segment的掩码，初始化如下：
```
        int segmentShift = 0;

        int segmentCount;
        for(segmentCount = 1; segmentCount < this.concurrencyLevel && (!this.evictsBySize() || (long)(segmentCount * 20) <= this.maxWeight); segmentCount <<= 1) {
            ++segmentShift;
        }

        this.segmentShift = 32 - segmentShift;
        this.segmentMask = segmentCount - 1;
        this.segments = this.newSegmentArray(segmentCount);
        int segmentCapacity = initialCapacity / segmentCount;
        if(segmentCapacity * segmentCount < initialCapacity) {
            ++segmentCapacity;
        }
```
可以看出segment的偏移量和掩码与设置的`concurrencyLevel`缓存并发级别和`maxWeight`缓存容量有关。
这样就将table均匀的放在不同的segment中，最大限度的避免哈希冲突。更多可参见[这里](http://ifeve.com/concurrenthashmap/)。

## 线程安全的cache put、get操作
 cache put操作如下：

 * 通过`key`的`hash`值找到相应的`Segment`；
 * 加锁，其他线程的put操作无法更新该`Segment`，但是可以更新其他`segment`；
 * 查找是否已有`key`存在，若存在则更新；若不存在，则新建`entry`，挂接到已有的`entry`链上；
 * 更新节点数量；
 * 解锁，其他线程可更新该`Segment`；
 
 基于超时机制的cache get操作如下：

* 通过`key`的`hash`值找到相应的`Segment`；
* 通过key获取到未过期的`entry`，若有，则到第`3`步；若没有，则到第`4`步；
* 则根据`key`获取未过期的`value`，若有，则直接返回；若没有，则到第`4`步；
* 判断是否有其他线程在`get`中，若有，则阻塞等待，直到其他线程返回结果；若没有，则加载新数据，loading...

>在cache get操作中并没有对所操作的`segment`加锁，那会不会获取到的是旧值，存在数据的不一致呢？不会，因为`ReferenceEntry`中`value`定义如下：

```
volatile ValueReference<K, V> valueReference
```
定义为`volatile`之后，保证了其`内存可见性`，这样其他线程无论何时都能得到最新的值，在保证线程安全的前提下又最大限度的提高了并发性能。

## 动态加载
动态加载行为发生在获取不到数据或者是数据已经过期的时间点，Guava动态加载使用回调模式，用户自定义加载方式，然后Guava cache在需要加载新数据时会回调用户的自定义加载方式：
```
segmentFor(hash).get(key, hash, loader);
```
`loader`即为用户自定义的数据加载方式，当某一线程get不到数据并且`没有其他线程在get数据`时，会去回调该自定义加载方式去加载数据。

在平常的开发中用到缓存的地方一般会做如图操作：
![0_1478862263360_缓存更新.png](/uploads/files/1478862263500-缓存更新-resized.png) 
这样的缓存处理方式在高并发场景并且缓存要重启或者是缓存失效时会有大量线程集中访问DB，给DB造成压力，也就是缓存雪崩。在Guava cache的超时模式下采用了以下方式：
![0_1478862280360_缓存操作2.png](/uploads/files/1478862280516-缓存操作2-resized.png) 

这样在某一线程get不到数据时会先检查是否有其他线程在试图get数据，如果有，则阻塞等待其他线程的结果数据，若没有，则去load数据，其他get线程会阻塞等待。

### 参考

[并发编程网](http://ifeve.com)

[Guava LocalCache 缓存介绍及实现源码深入剖析](http://ju.outofmemory.cn/entry/262684)

[美团技术博客](http://tech.meituan.com/avalanche-study.html)

并发编程实践

Java并发编程的艺术
