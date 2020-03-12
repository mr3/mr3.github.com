---
layout:       post
title:        Memory leaks by override finalize method
category:     tech
description:  Memory leaks by override finalize method, Zstd
---

## Summary
兄弟团队使用了[Zstd-jni](https://github.com/luben/zstd-jni), 一款提供快速，高性能压缩无损算法的类库，并推荐使用，

我在使用的过程中发现存在内存泄漏，原因是因为类重写了finalize方法，

我也在[issue#83:finalize() should not be overridden](https://github.com/luben/zstd-jni/issues/83)反馈了此问题，

作者已经在[v.1.4.4-3版本:Add option to skip finalizers](https://github.com/luben/zstd-jni/commit/2dd134c987732cca468f3fee8eb50d9c6bb149e0) 增加了补丁，

本文记录发现并解决的过程。

## Code
```Java
byte[] compress(String obj, String encoding) throws IOException {
    try (ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream(1024);
        OutputStream outputStream = new ZstdOutputStream(byteArrayOutputStream)) {
        outputStream.write(obj.getBytes(encoding));

        outputStream.flush();
        outputStream.close();
        return byteArrayOutputStream.toByteArray();
    } catch (IOException e) {
        e.printStackTrace();
        throw e;
    }
}
```

## Monitor and analysis
机器在发布Baking过程中通过监控发现堡垒机Survivor区和OldGen都出现异常曲线，如下图
<p><img src="/img/posts/tech/hickwall_memory_leaks_mosaic.jpg" alt="hickwall_memory_leaks_mosaic" width="850"></p>

拉Dump使用MAT分析定位到原因是在[ZstdOutputStream]对象，对象引用关系如下图

![ZstdOutputStream.mat_dominator_tree](/img/posts/tech/ZstdOutputStream.mat_dominator_tree.png)

## Fix
定位到问题以后我们解决的方法也很简单，

就是新建一个子类ZstdExtendOutputStream继承ZstdOutputStream，

重写finalize，保持方法为空，简要如下
```java
public class ZstdExtendOutputStream extends ZstdOutputStream {
    // Omitted code...

    @Override
    protected void finalize() throws Throwable {
    }
}
```

## Final
谨记：在使用到开源公共类库的时候，一定要做到充分的测试才可以使用。

---

## Concept
在计算机科学中，内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。内存泄漏通常情况下只能由获得程序源代码的程序员才能分析出来。

## Result
内存泄漏会因为减少可用内存的数量从而降低计算机的性能。最终，在最糟糕的情况下，过多的可用内存被分配掉导致全部或部分设备停止正常工作，或者应用程序崩溃。

## Dump
>jmap -dump:(dump-options) 示例：jmap -dump:format=b,file=zstd-compress.hprof 567280
> - format=b binary format
> - file=(file) dump heap to (file)
> - pid 进程ID

## Troubleshooter tool
[Memory Analyzer (MAT)](https://www.eclipse.org/mat/)

The Eclipse Memory Analyzer is a fast and feature-rich Java heap analyzer that helps you find memory leaks and reduce memory consumption.

Use the Memory Analyzer to analyze productive heap dumps with hundreds of millions of objects, quickly calculate the retained sizes of objects, see who is preventing the Garbage Collector from collecting objects, run a report to automatically extract leak suspects.

## Source code
- [Object.finalize](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#finalize--)
- [Finalizer class#FinalizerThread](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/ref/Finalizer.java#l186)

## Not recommended
1. 不及时，迟早会被回收
2. 资源可能用尽，导致系统崩溃
3. 性能损耗，JVM需要执行更多操作，检测重写的Finalize方法，创建 Finalizer
4. finalize 执行异常会被忽略
5. 系统移植差

## Alternative solution
- [AutoCloseable](https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html)
- [Closeable](https://docs.oracle.com/javase/8/docs/api/java/io/Closeable.html)

## Reference

### English
- [A Guide to the finalize Method in Java](https://www.baeldung.com/java-finalize)
- [Understanding Memory Leaks in Java](https://www.baeldung.com/java-memory-leaks)
- [Deprecate Object.finalize](https://bugs.openjdk.java.net/browse/JDK-8165641)
- [The try-with-resources Statement](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- [why-would-you-ever-implement-finalize](https://stackoverflow.com/questions/158174/why-would-you-ever-implement-finalize)

### 中文
- [JVM 源码分析之 FinalReference 完全解读](https://www.infoq.cn/article/jvm-source-code-analysis-finalreference)
- [你假笨在QCon上关于JVM性能问题分析的视频分享](https://mp.weixin.qq.com/s/OVtGfivZxBt8Ht2yZ8rccg)
- [深入理解Java try-with-resource](http://www.kissyu.org/2016/10/06/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%20try-with-resource/)
- [Java的Finalizer引发的内存溢出](https://www.cnblogs.com/benwu/articles/5812903.html)
- 深入理解Java虚拟机 第三章 垃圾收集器与内存分配策略
- [深入探索 JVM 自动资源管理](https://www.infoq.cn/article/Finalize-Exiting-Java)
- [Java 将弃用 finalize() 方法？](https://www.infoq.cn/article/2017/03/Java-Finalize-Deprecated/)
- [JVM故障分析系列之七：使用MAT的Histogram和Dominator Tree定位溢出源](https://www.javatang.com/archives/2017/11/08/11582145.html/comment-page-1)
