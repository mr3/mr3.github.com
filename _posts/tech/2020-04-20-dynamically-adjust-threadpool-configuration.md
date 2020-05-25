---
layout:       post
title:        动态调整线程池配置
category:     tech
description:  Dynamically adjust threadpool configuration, Java
tags:
    - threadpool
    - Java
---
## 简述

生产环境有遇到过需要动态调整线程池配置的需求, 
参考美团技术团队发表的文章[《Java线程池实现原理及其在美团业务中的实践》](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html),
自己撸了一版, 实践的过程中也遇到一些细节问题, 以此记录.

## 知识储备

如果对线程池使用不太了解的话, 强烈建议先看看上面提到的美团技术团队发表的文章. 请注意, 请注意, 是强烈建议.
 
如果有兴趣的话可以看看下面几个类的源码, 
* [ThreadPoolExecutor.java](http://hg.openjdk.java.net/jdk9/jdk9/jdk/file/65464a307408/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java)
* [LinkedBlockingQueue.java](http://hg.openjdk.java.net/jdk9/jdk9/jdk/file/65464a307408/src/java.base/share/classes/java/util/concurrent/LinkedBlockingQueue.java)
* [Executors.java](http://hg.openjdk.java.net/jdk9/jdk9/jdk/file/65464a307408/src/java.base/share/classes/java/util/concurrent/Executors.java)

## 调整线程池大小

调整线程池大小这个实现是非常简单的, 就是利用 `ThreadPoolExecutor` 的实例方法 `setCorePoolSize`,
`setMaximumPoolSize` 去调整`ThreadPoolExecutor`实例的核心和最大线程池大小. 
需要提一下的是 setCorePoolSize 在JDK 8是有Bug的, JDK 8 实现代码如下
```java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0)
        throw new IllegalArgumentException();
    // 省略无关代码
}
```

可以看到设置 `corePoolSize` 的时候并没有和 `maximumPoolSize` 进行比较, 
如果直接使用的话, 是允许设置`corePoolSize`大于`maximumPoolSize`的, 
这就和我们创建线程池的认知产生了误差.
如果直接使用, 后面升级了JDK的话, 会出现报错的情况.

此Bug在2014-05-21日 JDK 9 版本已经修复. JDK 9 修复代码如下,
仅仅是多加了一个判断条件 `maximumPoolSize < corePoolSize`

```java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0 || maximumPoolSize < corePoolSize)
        throw new IllegalArgumentException();
    // 省略无关代码
}
```

所以我们这里为了兼容以后升级JDK版本, 不用再去修改代码, 
修改`corePoolSize`和`maximumPoolSize`的时候需要增加判断, 伪代码如下
```java
public void resetThreadPoolSize() {
    if (newCorePoolSize > executor.getMaximumPoolSize()) {
        executor.setMaximumPoolSize(newMaximumPoolSize);
        // After jdk8 will throw exception if greater than the maximum pool size
        executor.setCorePoolSize(newCorePoolSize);
    } else {
        executor.setCorePoolSize(newCorePoolSize);
        executor.setMaximumPoolSize(newMaximumPoolSize);
    }
}
```

Bug Details: [ThreadPoolExecutor's setCorePoolSize method allows corePoolSize > maxPoolSize](https://bugs.openjdk.java.net/browse/JDK-7153400)

Change Log: [http://hg.openjdk.java.net/jdk9/jdk9/jdk/rev/eb6f07ec4509](http://hg.openjdk.java.net/jdk9/jdk9/jdk/rev/eb6f07ec4509)


## 调整线程池队列容量

不同的场景使用不同的线程池, 我们有一些需求会使用到fixed thread pool,
也就是固定线程池大小, 固定容量的线程池. 譬如: 写日志. 低延迟, 可以容忍丢失.

`Executors.newFixedThreadPool()`默认实现队列使用的是`LinkedBlockingQueue`, 
这是一个可选容量的阻塞队列, 我们通常都是需要设置 capacity 的, 为了防止队列长度过大导致OOM,
因为默认容量是 2<sup>31</sup>-1. `LinkedBlockingQueue`的 capacity 如下

```java
/** The capacity bound, or Integer.MAX_VALUE if none */
private final int capacity;
```

看代码我们知道`capacity`是加了final关键字的, 初始化后无法改变,
如果我们想要修改它, 就必须自己实现一个可变容量的阻塞队列.

这里我们的实现方式参考`rabbitmq-java-client`项目中的 [VariableLinkedBlockingQueue.java](https://github.com/rabbitmq/rabbitmq-java-client/blob/master/src/main/java/com/rabbitmq/client/impl/VariableLinkedBlockingQueue.java)
容量修改的实现方法如下
```java
/**
 * Set a new capacity for the queue. cao'c
 * @param capacity the new capacity for the queue
 */
public void setCapacity(int capacity) {
    final int oldCapacity = this.capacity;
    this.capacity = capacity;
    final int size = count.get();
    if (capacity > size && size >= oldCapacity) {
        signalNotFull();
    }
}
```
根据代码我们可以知道, 如果是增加容量超过当前队列大小,
则需要通知之前因为队列满而阻塞的调用, 使其成功执行.
另外要注意的就是如果队列容量由大变小, 
`put/offer/take/poll/remove/clear/drainTo`里面涉及到`== capacity`的判断条件需要改为 `>= capacity`.

## 总体设计

核心代码梳理完成后, 最终加上分布式配置+监控埋点+实时报警去修改线程池实例参数, 设计如下图

![arch design](/img/posts/tech/dynamically-adjust-threadpool-configuration-arch-design.png)

## 监控报警

发布测试环境后, 监控效果图如下, 当然你可以根据自己的实际情况去设置合理的报警阈值

![thread monitor](/img/posts/tech/threadpool-monitor.png)

## 总结

业务场景中碰到过太多次因为调整线程池需要重新发布应用的情况, 
所以动态调整线程池配置对开发来说是非常有必要的,
另外开发对应用线程池资源的使用情况也需要监控和报警,
这样一旦出现类似的故障, 就可以第一时间处理,
而不是无从下手, 或者需要重新发布应用导致故障影响范围增加.

## 优化

通过字节码注入方式实现, 而不在需要封装一个Jar包的形式供客户端使用. 
做到让客户端使用无感知, 这是我们优化的方向.
