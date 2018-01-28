---
title: hdfs&mapreduce测试之wordCount
date: 2017-12-31 18:14:08
tags:
- hadoop
categories:
- hadoop
---
<!--# 02 - hdfs&mapreduce测试之wordCount-->
<!--more-->
## 一、测试HDFS
hadoop启动成功后，首先通过``http://主机地址:50070``到web界面，查看一下HDFS中的文件，找到菜单栏 Browse the file system
![](http://wx2.sinaimg.cn/large/005TBZ5oly1fnnk2dnae9j30mw0363ye.jpg)  

进入之后，可以看到根目录 / 下面没任何文件，可以通过``hadoop fs -put <linux上的文件> <hdfs上的路径>``上传一个文件到hdfs文件系统中，这里先上传一个文件感受一下。  

上传jdk到hdfs根目录下``hadoop fs -put jdk-7u65-linux-i586.tar.gz hdfs://master:9000/``  
![](http://wx4.sinaimg.cn/large/005TBZ5oly1fnnleok1cjj311u09adg3.jpg)

上传成功，可以看到文件的基本信息。hdfs可以上传文件，同样可以下载文件，web界面通过点击文件名下载到本地windows。  
Linux通过``hadoop fs -get <hdfs上的路径> <linux路径>``  

下载jdk到本地 /home/hadoop 目录下  
```shell
[hadoop@master ~]$ pwd
/home/hadoop
[hadoop@master ~]$ ll
total 8
drwxrwxr-x. 4 hadoop hadoop 4096 Jan 20 01:49 tmp
drwxrwxr-x. 4 hadoop hadoop 4096 Jan 21 01:48 tools
[hadoop@master ~]$ hadoop fs -get hdfs://master:9000/jdk-7u65-linux-i586.tar.gz /home/hadoop/
[hadoop@master ~]$ ll
total 140232
-rw-r--r--. 1 hadoop hadoop 143588167 Jan 21 01:51 jdk-7u65-linux-i586.tar.gz
drwxrwxr-x. 4 hadoop hadoop      4096 Jan 20 01:49 tmp
drwxrwxr-x. 4 hadoop hadoop      4096 Jan 21 01:48 tools
```
看到成功下载jdk到本地，表示hdfs正常工作，接下来测试一下mapreduce  

## 二、测试MapReduce  
测试 MapReduce 要通过写一些程序测试，这里可以先通过 hadoop 自带的一些例子程序来测试。  
这些例子程序在``hadoop安装目录/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.x.x.jar``中，这个jar包里面有一些例子程序，其中就包含统计单词个数的 WordCount  

接下来就通过WordCount来实现一个小例子  
* 新建一个文件testCount.txt，输入一些单词：  
```shell
[hadoop@master mapreduce]$ vi testCount.txt
hello world
hello hadoop
hello hdfs
hello mapreduce
hello yarn
```
* mapreduce是在集群上面运行，所以数据也要放到集群上面  
将testCount.txt上传到hdfs中，为了方便，在根目录下新建一个文件夹 wordcount/input 来存放数据

```shell
[hadoop@master mapreduce]$ hadoop fs -mkdir hdfs://master:9000/wordcount
[hadoop@master mapreduce]$ hadoop fs -mkdir hdfs://master:9000/wordcount/input
```   
可以通过web界面查看文件夹</br>
![](http://wx2.sinaimg.cn/large/005TBZ5oly1fnnnxthe1yj30zn08tglt.jpg)</br>  

* 上传testCount.txt到input文件夹中:  
``[hadoop@master mapreduce]$ hadoop fs -put testCount.txt /wordcount/input``  

* 运行 hadoop-mapreduce-examples.jar中的wordcount程序，注意wordcount后面要跟两个参数，**/wordcount/input** 数据输入源，  **/wordcount/output** 数据输出文件夹，运行成功后数据输出的地方，可自定义位置  
``hadoop jar hadoop-mapreduce-examples-2.4.1.jar wordcount /wordcount/input /wordcount/output``  
```shell
18/01/21 03:26:20 INFO mapreduce.Job:  map 0% reduce 0%
18/01/21 03:26:27 INFO mapreduce.Job:  map 100% reduce 0%
18/01/21 03:26:33 INFO mapreduce.Job:  map 100% reduce 100%
18/01/21 03:26:34 INFO mapreduce.Job: Job job_1516452484789_0003 completed successfully
18/01/21 03:26:34 INFO mapreduce.Job: Counters: 49
```  
运行成功后，查看输出文件夹，里面多了两个文件  
```shell
[hadoop@master mapreduce]$ hadoop fs -ls /wordcount/output
Found 2 items
-rw-r--r--   1 hadoop supergroup          0 2018-01-21 03:26 /wordcount/output/_SUCCESS
-rw-r--r--   1 hadoop supergroup         51 2018-01-21 03:26 /wordcount/output/part-r-00000
```
查看输出的文件内容  
``hadoop fs -cat /wordcount/output/part-r-00000``
```shell
/wordcount/output/part-r-00000
hadoop  1
hdfs    1
hello   5
mapreduce       1
world   1
yarn    1
```
