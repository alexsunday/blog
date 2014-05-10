---
layout: post
title: Spark集群搭建与运行
tags: [Spark, Spark Standalone, Spark集群, Scala, pyspark]
keywords: Spark, Spark Standalone, Spark集群, Scala, pyspark
---

{{ page.title }}
================ 

#Spark集群搭建与运行
[前文](http://blog.ccgzs.org/2014/04/23/spark-bdas-learn-note.html)介绍了Spark在Win32平台下的配置与运行，本文续写Spark在Linux平台生产环境下的运行方式。Spark在Linux下支持多种不同的运行方式，各有优劣，如下介绍：

- 开发测试用的`local`模式
> 开发与测试专用，可运行于JDK所有适用的平台，比较适用于Win32平台开发所用，用`local`指代。但local模式支持指定线程数运行，如 **local[N]** 则为指定N个线程的方式运行。

- 与资源管理层`Mesos`协同运行
> Spark最初版本的开发时就考虑到了与资源管理层的协同，但时下流行的Hadoop Yarn并未诞生，从而选择了mesos这一平台，参考[此处](http://blog.csdn.net/pelick/article/details/14522447)获得Spark on Mesos的更多信息。

- 与资源管理器`YARN`协同运行
> 与Mesos相比，Spark on YARN并不成熟，国内互联网公司豆瓣使用Python，单独[Fork](https://github.com/douban/dpark)了[Spark](http://www.douban.com/subject/10774736/)，目前并不支持Yarn，可见Yarn的支持之延后。

- 单独的`Standalone`集群模式
> Spark学习初期或集群规模较小时，推荐此种方式，Spark可不依赖于任何第三方的资源管理层，独自开启Master-Slave的集群模式，此种方式运行安装简单，本文以此为例讲解。

##Spark及依赖包下载

Spark由Scala写成，下载[Scala](http://www.scala-lang.org/download/)，解压到`/opt/scala`。

scala由Java写成，故需安装Java运行时环境，下载[JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，解压到`/opt/java`。

Spark可以由源码编译，参考[此处](http://spark.apache.org/docs/latest/)的官方文档，但编译过程类似Maven，需要下载较多的第三方依赖库，鉴于国内恶劣的网络环境，不推荐使用源码编译的方式。此处推荐使用官网预编译版本，在其[下载页面](http://spark.apache.org/downloads.html)下载其pre-built版本，如下图所示之处：

![Spark下载]({{ site.baseurl }}images/spark/pre_built.png)

此处下载其Hadoop2，CDH5的版本，[链接](http://d3kbcqa49mib13.cloudfront.net/spark-0.9.1-bin-hadoop2.tgz)，此版本能与Hadoop2交互，很方便读取HDFS上的文件。下载完成后解压到`/opt/spark`目录。

##Linux操作系统安装与配置
假设此处有5台机器用于安装Spark集群，配置各机器网络环境及hostname等，分别为:

- Master, 192.168.0.191, Hostname:ue191, user:spark
- Slave,  192.168.0.192, Hostname:ue192, user:spark
- Slave,  192.168.0.193, Hostname:ue193, user:spark
- Slave,  192.168.0.194, Hostname:ue194, user:spark
- Slave,  192.168.0.195, Hostname:ue195, user:spark

首先在所有的节点上新增Spark用户，配置其密码，生产环境请不要使用如此弱智的访问密码。

```bash
useradd spark
echo 123456|passwd spark --stdin
```

配置Master到各机器间的无密码访问，以下操作在Master下完成：

```bash
ssh-keygen
ssh-copy-id $USER@ue192
ssh-copy-id $USER@ue193
ssh-copy-id $USER@ue194
ssh-copy-id $USER@ue195
```

同时，需要`/opt/{java, scala, spark}`目录复制到其余各节点，这是体力活。完成后按如下形式修改`$HOME/.bash_profile`，配置环境变量。

``` bash
export JAVA_HOME=/opt/java
export SCALA_HOME=/opt/scala
export SPARK_HOME=/opt/spark
export PATH=$JAVA_HOME/bin:$SCALA_HOME/bin:$SPARK_HOME/bin:$PATH
export LD_LIBRARY_PATH=$JAVA_HOME/lib:$LD_LIBRARY_PATH
export CLASSPATH=$SPARK_HOME/conf:$SPARK_HOME/assembly/target/scala-2.10/spark-assembly_2.10-0.9.1-hadoop2.2.0.jar
```

留意，各机器均要按此方式修改。
##Spark集群的运行
首先是Master上，启动其Master节点，按如下方式:

```bash
cd $SPARK_HOME/sbin/
./start-master.sh
```

进入第一台Slave机器ue192，按如下方式启动slave节点：

```bash
cd $SPARK_HOME/sbin/
./start-slave.sh 1 spark://ue191:7077
```

进入第二台Slave机器ue193，按如下方式启动：

```bash
cd $SPARK_HOME/sbin/
./start-slave.sh 2 spark://ue191:7077
```

轮流替换start-slave.sh后的数字ID，启动其余各个节点即可，完成后可打开Master的Web UI，master:8080一探究竟.

##在集群上运行应用
最简单的在集群上运行应用的方式是指定MASTER环境变量后启动Spark自带的Shell即可，如下：

```
MASTER=spark://ue191:7077 spark-shell
```

接下来的操作就如前文所述即可体验集群下的运行速度了，个人感觉明显快过Mapreduce，太多。至于如何开发在Spark下运行的应用，留后文再述。

本文主要参考了[此处](http://cn.soulmachine.me/blog/20140130/)的教程才得以成功搭建Spark Standalone 集群环境，感谢。

