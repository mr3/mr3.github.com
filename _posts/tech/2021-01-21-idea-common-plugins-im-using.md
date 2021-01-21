---
layout:       post
title:        我常用的 IDEA 插件
category:     tech
description:  IDEA common plugins I am using
tags:
    - IDEA
    - Java
---

记录一下我常用的 IDEA 插件. 按照字母排序

### Alibaba Java Coding Guidelines

阿里巴巴 Java 编码准则, 无需多解释, 必装. [查看插件地址](https://plugins.jetbrains.com/plugin/10046-alibaba-java-coding-guidelines)

### Arthas IDEA

生成 Arthas 常用命令的插件, 再也不需要记命令拉. 一键生成, 简单易用. [查看插件地址](https://plugins.jetbrains.com/plugin/13581-arthas-idea)

[Arthas](https://arthas.aliyun.com/) 是 Alibaba 开源的 Java 应用诊断利器. 比较好用的功能有以下几点
- 快速定位应用的热点, 生成火焰图
- 在线排查问题, 无需重启, 热更新代码
- 查看函数调用情况, 包括调用链, 耗时, 出入参, 异常信息等
- 实时监控 JVM 运行状态

### Choose Runtime

更改 IDEA 启动的 Java Runtime, 除非必须否则不建议修改, [查看插件地址](https://plugins.jetbrains.com/plugin/12836-choose-runtime).

### Error Prone Compiler

插件通过使用 [Error Prone Java compiler](https://errorprone.info/)可以增强编译器的类型分析, 在编译时捕获常见的 Java 错误. 

Error Prone 是 Google 内部 Java 构建系统使用的 Java 静态分析工具, 可在编译时捕获常见的 Java 错误. 目前已开源. [查看插件地址](https://plugins.jetbrains.com/plugin/7349-error-prone-compiler)


类似的 Java 代码静态字节码分析插件还有 [SpotBugs](https://plugins.jetbrains.com/plugin/14014-spotbugs)

### Lombok

支持 [Project Lombok](https://projectlombok.org/) 的插件, Project Lombok 是一个Java增强类库, 

可以在编译期根据自己的注解处理器, 动态修改 [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) 增加新的节点, 最后头通过分析生成字节码,

你只需要添加一些简单的注解就可以让你省去大量的时间去编写 getter/setter/equals/ 等冗长代码, 

[查看目前支持的注解请](https://projectlombok.org/features/all), IDEA 2020.3 已经集成了 Lombok. [查看插件地址](https://plugins.jetbrains.com/plugin/6317-lombok).

### Maven Helper

如果你还在为解决 maven 包冲突而浪费时间, 你一定需要这个插件. 

**插件提供了 UI 界面分析解决 maven 包冲突, 操作极其容易上手.**

隔壁村的狗看一遍都会了, [查看插件地址](https://plugins.jetbrains.com/plugin/7179-maven-helper).

### PlantUML integration

IDEA 集成的查看 PlantUML 文件的插件, [PlantUML](https://plantuml.com/) 是一个开源工具, 

你可以使用纯文本语言生成 UML 图, 做设计的时候比较好用拉. [查看插件地址](https://plugins.jetbrains.com/plugin/7017-plantuml-integration)


### SequenceDiagram

生成代码时序图的插件, 对了解项目代码调用流程非常有帮助, 特别是当你接触到一个完全陌生的项目时. 

使用此插件 你还可以将时序图导出成图片或 PlantUML 文件, [查看插件地址](https://plugins.jetbrains.com/plugin/8286-sequencediagram).


### End

工具无他, 唯提高效率而已.