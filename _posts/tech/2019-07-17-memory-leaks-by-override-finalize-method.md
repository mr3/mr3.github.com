---
layout:       post
title:        重写finalize方法引发的内存泄露
category:     tech
description:  Memory leaks by override finalize method, Zstd
tags:
    - hotspot
    - JVM
    - Troubleshooting
---

## 简述
兄弟团队使用了[Zstd-jni](https://github.com/luben/zstd-jni), 
一款提供快速，高性能压缩无损算法的类库，在内部做了推荐.

我在`JDK 8`上使用的过程中发现存在内存泄漏，原因是因为`ZstdInputStream`,`ZstdOutputStream`重写了finalize方法.

我也在 [issue#83:finalize() should not be overridden](https://github.com/luben/zstd-jni/issues/83) 反馈了此问题，
作者已经在[v.1.4.4-3版本:Add option to skip finalizers](https://github.com/luben/zstd-jni/commit/2dd134c987732cca468f3fee8eb50d9c6bb149e0) 增加了补丁，

本文记录发现并解决的过程.

## 定位原因
在发布 Baking 过程中通过监控发现堡垒机`Survivor`区和`OldGen`都出现异常曲线，如下图
![hickwall_memory_leaks_mosaic](/img/posts/tech/hickwall_memory_leaks_mosaic.jpg)

拉Dump使用[Memory Analyzer (MAT)](https://www.eclipse.org/mat/)分析定位到原因是在[ZstdOutputStream](https://github.com/luben/zstd-jni/blob/master/src/main/java/com/github/luben/zstd/ZstdOutputStream.java)对象，对象引用关系如下图

![ZstdOutputStream.mat_dominator_tree](/img/posts/tech/ZstdOutputStream.mat_dominator_tree.png)

## 分析修复
首先我们要了解一下重写了finalize方法后, 虚拟机是怎么识别并执行的.

主线程启动调用构造函数初始化时通过方法名和签名查找register方法去初始化`_finalizer_register_cache`,
后面调用register会用到此对象

```cpp
// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/vmSymbols.hpp#l402
/* non-intrinsic name/signature pairs: */                                 
template(register_method_name,                      "register")
do_alias(register_method_signature,         object_void_signature) 


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/memory/universe.hpp#l148
static LatestMethodCache* _finalizer_register_cache; // static method for registering finalizable objects


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/memory/universe.cpp#l1093
// Setup static method for registering finalizers
// The finalizer klass must be linked before looking up the method, in
// case it needs to get rewritten.
InstanceKlass::cast(SystemDictionary::Finalizer_klass())->link_class(CHECK_false);
// 通过方法名和签名查找register方法
Method* m = InstanceKlass::cast(SystemDictionary::Finalizer_klass())->find_method(
                              vmSymbols::register_method_name(),
                              vmSymbols::register_method_signature());
if (m == NULL || !m->is_static()) {
    // 初始化_finalizer_register_cache, 通过日志可以看到对应Java代码 Finalizer.register
    tty->print_cr("Unable to link/verify Finalizer.register method");
    return false; // initialization failed (cannot throw exception yet)
}
Universe::_finalizer_register_cache->init(SystemDictionary::Finalizer_klass(), m);
```

类加载时, 类文件解析器classFileParser.cpp 解析方法(parse_method)时`判断类是否包含一个名为finalize并且返回void的非空方法`,
如果包含就在填充类实例(fill_instance_klass)时进行原子操作,采用按位或运算设置`JVM_ACC_HAS_FINALIZER`标识,
标记位包含finalized的类

```cpp
// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/vmSymbols.hpp#l318
template(finalize_method_name,                      "finalize") 


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/classFileParser.cpp#l2503
if (name == vmSymbols::finalize_method_name() &&
    signature == vmSymbols::void_method_signature()) {
  if (m->is_empty_method()) {
    _has_empty_finalizer = true;
  } else {
    _has_finalizer = true;
  }
}


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/classFileParser.cpp#l4284
// Check if this klass has an empty finalize method (i.e. one with return bytecode only),
// in which case we don't have to register objects as finalizable
if (!_has_empty_finalizer) {
  if (_has_finalizer ||
      (super != NULL && super->has_finalizer())) {
    k->set_has_finalizer();
  }
}


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/klass.hpp#l540
bool has_finalizer() const            { return _access_flags.has_finalizer(); }
bool has_final_method() const         { return _access_flags.has_final_method(); }
void set_has_finalizer()              { _access_flags.set_has_finalizer(); }


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/utilities/accessFlags.hpp#l62
JVM_ACC_HAS_FINALIZER = 0x40000000,     // True if klass has a non-empty finalize() method
bool has_finalizer() const { return (_flags & JVM_ACC_HAS_FINALIZER          ) != 0; }
void set_has_finalizer()             { atomic_set_bits(JVM_ACC_HAS_FINALIZER);  }
```
new Object() 创建对象时, 解释器调用`InterpreterRuntime::_new`去创建实例, 
通过`allocate_instance`计算对象大小分配内存, 
这个时候会检测是否需要注册Finalizer实例, 

RegisterFinalizersAtInit 默认为true, 是不会注册的,
如果用户自定义了JVM参数, 设置了[-XX:-RegisterFinalizersAtInit](https://club.perfma.com/topic/RegisterFinalizersAtInit) 为false,
则会在对象分配之后注册注册Finalizer实例.

查看所有JVM可以设置的参数列表可通过:`java -XX:+PrintFlagsFinal -version`

```cpp
// http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp#l147
JRT_ENTRY(void, InterpreterRuntime::_new(JavaThread* thread, ConstantPool* pool, int index))
  Klass* k = pool->klass_at(index, CHECK);
  InstanceKlass* klass = InstanceKlass::cast(k);

  // Make sure we are not instantiating an abstract klass
  klass->check_valid_for_instantiation(true, CHECK);

  // Make sure klass is initialized
  klass->initialize(CHECK);

  // At this point the class may not be fully initialized
  // because of recursive initialization. If it is fully
  // initialized & has_finalized is not set, we rewrite
  // it into its fast version (Note: no locking is needed
  // here since this is an atomic byte write and can be
  // done more than once).
  //
  // Note: In case of classes with has_finalized we don't
  //       rewrite since that saves us an extra check in
  //       the fast version which then would call the
  //       slow version anyway (and do a call back into
  //       Java).
  //       If we have a breakpoint, then we don't rewrite
  //       because the _breakpoint bytecode would be lost.
  oop obj = klass->allocate_instance(CHECK);
  thread->set_vm_result(obj);
JRT_END


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/instanceKlass.cpp#l1096
instanceOop InstanceKlass::allocate_instance(TRAPS) {
  bool has_finalizer_flag = has_finalizer(); // Query before possible GC
  // 计算对象大小
  int size = size_helper();  // Query before forming handle.

  KlassHandle h_k(THREAD, this);

  instanceOop i;

  i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
  // 检测是否需要注册Finalizer实例
  if (has_finalizer_flag && !RegisterFinalizersAtInit) {
    i = register_finalizer(i, CHECK_NULL);
  }
  return i;
}
```
Java中的类都是直接或者间接继承Object, 防止在 Object.init 的时候都去`register_finalizer`,
[JVMTI](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#whatIs) 可能会为Object.init重写字节码,
会将构造函数的return指令重写为`_return_register_finalizer`,

如果用户修改了`RegisterFinalizersAtInit` 为 false, 这里将不会发生重写.

重写目的是准确地遵循标准并在正确的时间处理该指令注册Finalizer实例.

JDK开发人员关于重写指令沟通邮件有兴趣可以看看: [Compiling Object.init](http://mail.openjdk.java.net/pipermail/hotspot-dev/2009-February/001385.html)

```cpp
// http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/rewriter.cpp#l497
if (RegisterFinalizersAtInit && _klass->name() == vmSymbols::java_lang_Object()) {
  bool did_rewrite = false;
  int i = _methods->length();
  while (i-- > 0) {
    Method* method = _methods->at(i);
    if (method->intrinsic_id() == vmIntrinsics::_Object_init) {
      // rewrite the return bytecodes of Object.<init> to register the
      // object for finalization if needed.
      methodHandle m(THREAD, method);
      rewrite_Object_init(m, CHECK);
      did_rewrite = true;
      break;
    }
  }
  assert(did_rewrite, "must find Object::<init> to rewrite it");
}
```

重写指令后, 字节码解释器处理`_return_register_finalizer`指令时,
判断是否`has_finalizer()`, 决定是否调用`InterpreterRuntime::register_finalizer`,
最终`InstanceKlass::register_finalizer`注册Finalizer实例


```cpp
// http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/bytecodeInterpreter.cpp#l1662
CASE(_return_register_finalizer): {
  oop rcvr = LOCALS_OBJECT(0);
  VERIFY_OOP(rcvr);
  // 如果类访问标识包含了JVM_ACC_HAS_FINALIZER, 则执行 InterpreterRuntime::register_finalizer
  if (rcvr->klass()->has_finalizer()) {
    CALL_VM(InterpreterRuntime::register_finalizer(THREAD, rcvr), handle_exception);
  }
  goto handle_return;
}


// http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp#l222
IRT_ENTRY(void, InterpreterRuntime::register_finalizer(JavaThread* thread, oopDesc* obj))
  assert(obj->is_oop(), "must be a valid oop");
  assert(obj->klass()->has_finalizer(), "shouldn't be here otherwise");
  // 执行 InstanceKlass::register_finalizer
  InstanceKlass::register_finalizer(instanceOop(obj), CHECK);
IRT_END
```


注册Finalizer实例时会使用之前已经init的_finalizer_register_cache, 最终执行`JavaCalls::call`
调用 Java 方法[Finalizer.register](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/ref/Finalizer.java#l85)

```cpp
// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/memory/universe.hpp#l301
static Method*      finalizer_register_method()     { return _finalizer_register_cache->get_method(); }


// http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/instanceKlass.cpp#l1081
instanceOop InstanceKlass::register_finalizer(instanceOop i, TRAPS) {
  if (TraceFinalizerRegistration) {
    tty->print("Registered ");
    i->print_value_on(tty);
    tty->print_cr(" (" INTPTR_FORMAT ") as finalizable", (address)i);
  }
  instanceHandle h_i(THREAD, i);
  // Pass the handle as argument, JavaCalls::call expects oop as jobjects
  JavaValue result(T_VOID);
  JavaCallArguments args(h_i);
  // 调用 Universe::finalizer_register_method, 最终执行 JavaCalls::call
  (THREAD, Universe::finalizer_register_method());
  JavaCalls::call(&result, mh, &args, CHECK_NULL);
  return h_i();
}

// http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/ref/Finalizer.java#l85
/* Invoked by VM */
static void register(Object finalizee) {
    new Finalizer(finalizee);
}
```

看到这里我们基本对虚拟机识别和执行`finalize`方法有一个大致的了解, 下面我们看一下`Finalizer`类的 Java 实现

其中注册为Finalizer对象的实例会被加到 ReferenceQueue 里, 也就是说它并不是立刻在垃圾回收周期中被回收
```java
private static ReferenceQueue<Object> queue = new ReferenceQueue<>();
```
queue 消费是通过守护线程`FinalizerThread`, 通过代码可以看到此线程创建时优先级被做了降级,
意味着CPU紧张时此线程可能不会被调度到
```java
static {
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
         tgn != null;
         tg = tgn, tgn = tg.getParent());
    Thread finalizer = new FinalizerThread(tg);
    finalizer.setPriority(Thread.MAX_PRIORITY - 2);
    finalizer.setDaemon(true);
    finalizer.start();
}
```
GC 时该线程从队列中移除并获取Finalizer实例, 执行实例方法`runFinalizer`

```java
private static class FinalizerThread extends Thread {
    private volatile boolean running;
    FinalizerThread(ThreadGroup g) {
        super(g, "Finalizer");
    }
    public void run() {
        if (running)
            return;

        // Finalizer thread starts before System.initializeSystemClass
        // is called.  Wait until JavaLangAccess is available
        while (!VM.isBooted()) {
            // delay until VM completes initialization
            try {
                VM.awaitBooted();
            } catch (InterruptedException x) {
                // ignore and continue
            }
        }
        final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
        running = true;
        for (;;) {
            try {
                Finalizer f = (Finalizer)queue.remove();
                f.runFinalizer(jla);
            } catch (InterruptedException x) {
                // ignore and continue
            }
        }
    }
}
```

`runFinalizer`最终还是调用Finalizer类包装对象的finalize方法,
finalize方法执行后`对象会在下次 GC 时被真正地回收`,
`runFinlizer`只会被执行一次, `remove` 之后Finalizer对象指向已经发生变化,
执行时会忽略对象finalize方法产生的异常

```java
private void runFinalizer(JavaLangAccess jla) {
    synchronized (this) {
        if (hasBeenFinalized()) return;
        remove();
    }
    try {
        Object finalizee = this.get();
        if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
            jla.invokeFinalize(finalizee);

            /* Clear stack slot containing this variable, to decrease
               the chances of false retention with a conservative GC */
            finalizee = null;
        }
    } catch (Throwable x) { }
    super.clear();
}
```

到此, 我们就了解了为什么重写finalize方法会导致内存泄露, 定位到问题以后我们解决的方法也很简单，
就是新建一个子类ZstdExtendOutputStream继承ZstdOutputStream，
重写finalize，保持方法为空，代码如下
```java
public class ZstdExtendOutputStream extends ZstdOutputStream {
    // Omitted code...

    @Override
    protected void finalize() throws Throwable {
    }
}
```

## 总结

强烈不推荐重写finalize方法, 原因如下, 另外在[JDK 9 中 Object.finalize 已经被弃用](https://bugs.openjdk.java.net/browse/JDK-8165641)
1. 回收不及时，`迟早`会被回收
2. 资源可能用尽，导致系统崩溃
3. 性能损耗，JVM需要执行更多操作，检测重写的finalize方法，额外创建 Finalizer
4. finalize 执行异常会被忽略
5. 系统移植差

## 改进方案

如果要安全地关闭占有的资源, 推荐使用[Try-with-resources 语句（TWR）](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html),
示例代码如下
```java
try (java.util.zip.ZipFile zf = new java.util.zip.ZipFile(zipFileName);
     java.io.BufferedWriter writer = java.nio.file.Files.newBufferedWriter(outputFilePath, charset)){
    // your code
}
```
你可以在try语句中声明一个或多个资源, TWR 语句可以确保在语句末尾关闭每个资源,
任何实现`java.lang.AutoCloseable`接口的对象, 其中包括实现`java.io.Closeable`接口的对象,
都可以当作资源使用 TWR 语句.


## 引用

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
