---
layout:       post
title:        JMockit with JDK 11
category:     tech
description:  JMockit with JDK 11, Java 11
tags:
    - JDK11
    - Troubleshooting
---

## 简述

项目从 JDK 8 升级为 JDK 11的过程中,

由于项目中使用了 JMockit, 导致项目单元测试执行一直失败,

本文记录解决单元测试执行失败以及JMockit 升级的过程. 

## JDK 8升级为JDK 11, JMockit 1.37
项目使用 `JDK 8` 时, JMockit 使用版本为 `1.37`,

在写单元测试时, 一直使用 [@Tested, @Injectable](https://jmockit.github.io/tutorial/Mocking.html#tested) 来进行测试类初始化和注入测试类依赖,

在升级为JDK 11之后, 运行单元测试, IDEA 提示以下错误: 

`java.lang.IllegalStateException: Running on JDK 9 requires -javaagent:<proper path>/jmockit-1.n.jar or -Djdk.attach.allowAttachSelf`

查看 [maven.surefire.plugin](http://maven.apache.org/surefire/maven-surefire-plugin/index.html) 文档得知, 2.1 版本之后 插件执行时允许用户添加命令至JVM选项,

根据提示在 [properties.argLine](http://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html#argLine) 增加 `-Djdk.attach.allowAttachSelf`
```xml
<properties>
    // ...other properities
    <argLine>
        -Djdk.attach.allowAttachSelf
    </argLine>
</properties>
```

再次运行单元测试, 上面错误没有了, IDEA 又提示以下错误

```
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the class NestHost or NestMembers attribute

at java.instrument/sun.instrument.InstrumentationImpl.redefineClasses0(Native Method)
```

依据关键信息: `attempted to change the class NestHost or NestMembers attribute`,

得知此问题是由于 `JDK 11` 发布后的增强型提案 [JEP181: 嵌套控制访问](http://openjdk.java.net/jeps/181) 引发的.

查看 [jmockit github issue 关于 Java 11 的 issue](https://github.com/jmockit/jmockit1/issues?q=Java+11) 记录, 看到 [issue/534:JMockit on Java 11](https://github.com/jmockit/jmockit1/issues?q=Java+11) 相关讨论

其中一条关键的信息: rliesenfeld closed this in [c5fffc1](https://github.com/jmockit/jmockit1/commit/c5fffc121e36870025efed10d610477e865f747c) on 23 Sep 2018, 得知 JMockit Member 已经修复了此问题.

通过 [JMockit release notes#1.43](https://jmockit.github.io/changes.html#1.43) 也可以印证
- Deprecated the withArgThat(Matcher) method (available in the Expectations and Verifications classes), in favor of argument capturing (the withCapture(...) methods) and the with(Delegate) method.
- `Issues closed: #534.`

## JDK 11, JMockit 1.49

我们将 [JMockit 版本](https://mvnrepository.com/artifact/org.jmockit/jmockit) 升级为当前最新版本 1.49 后,

发现 `JMockit.class 被移除`, 之前在测试类增加的注解 `@RunWith(JMockit.class) 需要移除`.

清理干净后 重新执行单元测试, 发现 `@Tested` 修饰的类实例总是为 null, 

查看 [JMockit release notes#1.42](https://jmockit.github.io/changes.html#1.42), 

- `JMockit now requires the use of the "-javaagent" JVM initialization parameter. See Running tests with JMockit.`
- Dropped support for mock parameters in TestNG methods.
- Issues closed: #515, #542.

得知 1.42 之后 使用任何 JMockit API, 都需要使用 `-javaagent:<proper path>/jmockit.1.x.jar` 进行测试JVM初始化, 

这里需要将之前增加的 `-Djdk.attach.allowAttachSelf` 替换为 `-javaagent`, 至此问题解决.

```xml
<properties>
    // ...other properities
    <argLine>
        -javaagent:"${settings.localRepository}"/org/jmockit/jmockit/${jmockit.version}/jmockit-${jmockit.version}.jar
    </argLine>
</properties>
```

Maven 相关修改请参照: [4.1 Running tests from Maven](https://jmockit.github.io/tutorial/Introduction.html#maven), 这里不再拷贝.

## 总结

排查 JMockit的问题, JMockit release notes 一定要细看, 重点信息都在上面了.
