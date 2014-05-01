---
layout: post
title: Spark上手试玩
tags: Spark Hadoop Hadoop2 Shark pyspark spark-shell
---

{{ page.title }}
================ 

##简介
**Spark**最近比较红，关于spark的一些介绍可以看[这里](http://zh.wikipedia.org/zh-cn/SPARK)，或者[这里](http://www.csdn.net/article/2013-04-26/2815057-Spark-Reynold)。本文是第一次试玩Spark的一些笔记，伯克利大学有一个针对BDAS的在线教程，在[这里](http://ampcamp.berkeley.edu/big-data-mini-course)，本文相当于其[第一篇](http://ampcamp.berkeley.edu/big-data-mini-course/data-exploration-using-spark.html)的中文笔记。

![Alt text]({{ site.baseurl }}images/spark-logo.png)

##下载安装
Spark基于[Scala](http://zh.wikipedia.org/zh-cn/Scala)开发，而Scala又是一种基于JVM的编程语言，于是你至少需要安装Java与Scala。与Hadoop类似，Spark生产平台只建议使用Linux，但Win32平台可做为开发测试所用。鉴于大部分在线教程均以Linux为例，故本文针对Windows平台进行讲解，简述其安装过程及其Python REPL的使用步骤。由于Spark及其社区正处于快速发展之中，故预计本文内容将很快变得不合适，本文以当前spark官方发布的最新版Spark-0.9.11为例进行说明。

- 下载[JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，解压至你想要的安装目录，本文以D:\JVM\为例说明，将jdk解压至D:\JVM\，并重命名为jdk，形成D:\JVM\jdk\bin\java.exe 类似结构
- 下载[Scala](http://www.scala-lang.org/download/)，并解压至D:\JVM，重命名为 scala，类似D:\JVM\scala结构
- 下载[Spark](http://spark.apache.org/downloads.html)，留意下载[预编译版本](http://d3kbcqa49mib13.cloudfront.net/spark-0.9.1-bin-hadoop2.tgz)，不要尝试手工编译spark，国内网络环境往往以失败告终
- 配置以下环境变量
 - `JAVA_HOME=D:\\JVM\\jdk`
 - `SCALA_HOME=D:\\JVM\\scala`
 - `SPARK_HOME=D:\\JVM\\spark`
 - `PATH=%JAVA_HOME%\bin;%SCALA_HOME%\bin;%SPARK_HOME%\bin`

###下载维基百科流量统计数据集
伯克利大学的教程中，以维基百科的访问流量数据为例子进行讲解，但源数据放在Amazon 云盘上，但经测试，其免费版并不支持运行超过2个以上的Instance，故无法实际运行教程中的脚本。且目前数据量已达2.5TB之大，即使压缩后也多大800多GB，初学者很可能无太大动力下载原始文件，这里给出下载其一部分样例文件的方式。[此处](http://dumps.wikimedia.org/other/pagecounts-raw/2013/2013-12/)是维基百科2013年12月的一部分流量访问数据，全部共60多G，你可以选择性下载。

![Alt text]({{ site.baseurl }}images/wiki_data.png)

下载后就可以使用普通的文本编辑器打开，由于文件较大，推荐使用Linux/Unix的less命令查看，文件头N行[*]开头的是头信息区，其后是正文数据区域，其数据区域分为4个字段，如下所示：

```
zh ?page=http://www.stockphotosharing.com/Themes/Images/users_raw/id.txt 3 39267
en Main_Page 7 51309
en Special:Boardvote 1 11631
en Special:Imagelist 1 931
```

分别是 **语种名**，**页标题**，**点击数**，**页大小**，对比`AMPCAMP`的教程来说少了前面的日期时间字段，但由于文件本身文件名中有时间日期字段，故可以使用如下命令处理之，并合并。

``` Bash
tar xf pagecounts*.gz
for fn in `ls pagecounts*`
do
    sed -i '/^[*]/d' $fn
    dt=`echo $fn|awk -F'pagecounts-' '{print $2}'`
    sed -i 's/^/$dt /' $fn
done
cat pagecounts* >>_tmp_pagecounts
mv _tmp_pagecounts pagecounts
```

##交互式运行
设置完环境变量后，运行中输入cmd，敲开命令提示窗口，键入pyspark，如何，是否已经正确启动了spark实例，正确启动后，如下图所示：
![Alt text]({{ site.baseurl }}images/pyspark.png)

图中可以看到Spark的日志显示，有一行：

``` text
14/04/22 22:56:40 INFO SparkUI: Started Spark Web UI at http://HL-20140104AJLE:4040
```

提示我们，Spark在本机的4040端口打开了其Web监控界面，使用浏览器打开后将看到如下图所示：

![Alt text]({{ site.baseurl }}images/spark_ui.png)

最后一行内容告诉我们，Spark初始化完毕，我们可以使用sc这个Spark的上下文环境，接下来正式开始我们的Spark之旅，首先读取指定的pagecounts文件：

``` python
>>> sc
<pyspark.context.SparkContext object at 0x7f7570783350>
>>> pagecounts = sc.textFile("/path/to/pagecounts")
13/02/01 05:30:43 INFO mapred.FileInputFormat: Total input paths to process : 74
>>> pagecounts
<pyspark.rdd.RDD object at 0x217d510>
```

接下来我们简单的看下目标数据，通过调用SparkContext对象的take方法可以实现：

``` python
>>> pagecounts.take(10)
...
[u'20090505-000000 aa.b ?71G4Bo1cAdWyg 1 14463', u'20090505-000000 aa.b Special:Statistics 1 840',\
u'20090505-000000 aa.b Special:Whatlinkshere/MediaWiki:Returnto 1 1019',\
u'20090505-000000 aa.b Wikibooks:About 1 15719', u'20090505-000000 aa ?14mFX1ildVnBc 1 13205',\
u'20090505-000000 aa ?53A%2FuYP3FfnKM 1 13207', u'20090505-000000 aa ?93HqrnFc%2EiqRU 1 13199',\
u'20090505-000000 aa ?95iZ%2Fjuimv31g 1 13201',\
u'20090505-000000 aa File:Wikinews-logo.svg 1 8357', u'20090505-000000 aa Main_Page 2 9980']
```

使用python的for循环，简单浏览这10条数据

``` python
>>> for x in pagecounts.take(10):
...    print x
...
20090505-000000 aa.b ?71G4Bo1cAdWyg 1 14463
20090505-000000 aa.b Special:Statistics 1 840
20090505-000000 aa.b Special:Whatlinkshere/MediaWiki:Returnto 1 1019
20090505-000000 aa.b Wikibooks:About 1 15719
20090505-000000 aa ?14mFX1ildVnBc 1 13205
20090505-000000 aa ?53A%2FuYP3FfnKM 1 13207
20090505-000000 aa ?93HqrnFc%2EiqRU 1 13199
20090505-000000 aa ?95iZ%2Fjuimv31g 1 13201
20090505-000000 aa File:Wikinews-logo.svg 1 8357
20090505-000000 aa Main_Page 2 9980
```

现在我们正式进入Spark的分布式计算功能，首先我们看看此文件总共有多少行内容：

``` python
>>> pagecounts.count()
```

输入完指令并回车确认后，立即打开浏览器，刷新之前的Spark监控界面，可以发现Spark已经将提交了一项任务，并自动将任务分片开始执行，稍等片刻，控制台就有了最终的结果。

是否觉得这里的『分布式执行』并没有多少神秘，而且速度也未有想象中的神速？对的，因为我们只是单机运行，而且数据量本身并不是太大，而执行过程却仍然会分片、汇总（也就是map、reduce）等操作，故未能提现其效率优势，如若使用真正分布式运算，还是会极大的提升其运行效率的。

接下来我们使用比较pythonic的方式进行数据浏览与分析，首先我们简单统计下中文页面有几多，使用如下命令：

``` python
>>> zhPages = pagecounts.filter(lambda x: x.split(" ")[1].startswith("zh")).cache()
>>> zhPages.count()
```

想必这一行已经足够的『一目了然』了吧，lambda函数极大程度上的解放了手指的工作量，留意末尾的cache()调用，这意味着将数据进行内存缓冲，将加速下面的分析速度。

下面的代码将刚刚整理的中文数据，按日期进行汇总，相当于SQL操作的sum and group by，但是在pythonic看来却更加自然：

``` python
>>> enTuples = zhPages.map(lambda x: x.split(" "))
>>> zhKeyValuePairs = zhTuples.map(lambda x: (x[0][:8], int(x[3])))
>>> zhKeyValuePairs.reduceByKey(lambda x, y: x + y, 1).collect()
```

的确，如果你愿意，可以像书写SQL语句一样，将其放入同一行完成，一起来感受神奇：

``` python
>>> zhPages.map(lambda x: x.split(" ")).map(lambda x: (x[0][:8], int(x[3]))).reduceByKey(lambda x, y: x + y, 1).collect()
...
[(u'20090506', 204190442), (u'20090507', 202617618), (u'20090505', 207698578)]
```

以上只是Spark Python shell的简单用法，参考[原教程](http://ampcamp.berkeley.edu/big-data-mini-course/data-exploration-using-spark.html)以获得更多咨询。

当前Spark社区发展迅速，已经衍生出了类似Hadoop的一套完整的生态系统，原教程既是针对Spark生态[BDAS](https://amplab.cs.berkeley.edu/software/)的入门，英文好的朋友还是直接看原文好了。
