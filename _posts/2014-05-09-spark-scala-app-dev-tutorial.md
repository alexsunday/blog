---
layout: post
title: Spark应用开发流程简述
tags: [Spark, Scala, Bigdata, Java]
keywords: Spark, Scala, Bigdata, Java
description: 本文简述Spark-scala应用程序到开发步骤
published: false
---

Spark应用开发流程简述
================ 

Spark正当红，可参见[Spark上手试玩](http://blog.ccgzs.org/2014/04/23/spark-bdas-learn-note.html)，Spark的性能使得其成为了最耀眼的明星，其开发方式-scala与python吸引了不少大数据玩家。但由于Spark毕竟属新兴事务，网上不少教程要么过时，要么都是无责任copy，多数实例已无法正常运行，且Spark官方支持Java、Python、Scala三种编程语言，但只有scala为其实现语言，本文针对当前Spark最新版 0.9.1 ，简述spark-scala应用的常规开发流程。

##Java vs Scala
##源码编译 vs 预编译
大多网上流传教程均提及spark的从源码编译，鉴于国内网络环境普遍不佳
##项目工程设置
命令行开发一个spark应用，新建一个工程目录

```bash
mkdir test
sbt #进入sbt的命令行模式
set name := "MyJob"
set version := "0.1"
set scalaVersion := "2.10.4"
set libraryDependencies +=  "org.apache.spark" %% "spark-core" % "0.9.1"
set resolvers += "Akka Repository" at "http://repo.akka.io/releases/" 
session save
exit
```

##源文件即代码编辑
创建src/main/scala文件夹层级

```bash
mkdir src/main/scala -pv
```
##打包与运行
使用sbt打包，亦可直接使用sbt运行
```bash
sbt package
sbt run
```

spark cluster中运行
```bash
CLASSPATH=$SPARK_HOME/assembly/target/scala-2.10/spark-assembly_2.10-0.9.1-hadoop2.2.0.jar
java -cp :/opt/spark/conf:/opt/spark/assembly/target/scala-2.10/spark-assembly_2.10-0.9.1-hadoop2.2.0.jar:/home/spark/test/target/scala-2.10/simple-project_2.10-0.1.jar SimpleJob local
```
