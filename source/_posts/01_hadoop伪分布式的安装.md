---
title: 01 - Hadoop伪分布式的搭建
date: 2017-12-31 18:14:08
tags:
- hadoop
categories:
- hadoop
---

<!--# 01 - Hadoop伪分布式的搭建-->
<!--more-->
网上看了许多教程，参考着[官方文档](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)，按照步骤一步一步来，最后顺利在CentOS中成功安装Hadoop并运行，这里特做一个总结，下面就把详细的安装步骤叙述一下。  

## 安装环境
由于本人电脑配置不高，本文所使用的是 CentOS 6.5 32 位作为系统环境,也可以用其他版本的 Linux，例如: Ubuntu。  
基于 Hadoop2.4.1(stable) 版本，Hadoop2.X.X 同样适用，

hadoop 官网本来提供的都是32位，因为我们服务器中使用的大部分都是64位，所以不得不编译。后来官网从 hadoop2.5 版本开始就提供了64位。请根据自己的 Linux 系统，下载对应的 hadoop 版本，否则64位的 hadoop 运行到32位的Linux上就会出现一些问题，可到[官方网站](https://archive.apache.org/dist/hadoop/common/)下载对应的稳定版本。  

## 一、安装JDK
根据官方说明，Hadoop 版本大于等于2.7：要求 Java7(openjdk/oracle) 或者以上版本，hadoop版本小于等于2.6：要求至少为 Java 6(openjdk/oracle)，我们这里使用JDK1.7。  
JDK下载地址：http://www.oracle.com/technetwork/java/javase/downloads/index.html  

* **上传JDK到linux中**  
下载完成之后，将JDK上传到Linux中，这里推荐两种上传方式:  
**1.FileZilla**:免费小巧的FTP软件，安装之后功能简单明了，直接用鼠标拖动上传文件。  
**2.SecureCRT**:很好用的远程连接软件，上传文件只需要在SecureCRT连接服务器后，使用快捷键``Alt+P``打开sftp窗口  
使用命令``put g:/jdk-7u65-linux-i586.tar.gz``(上传文件所在的路径)  
个人喜欢使用第二种方式，一条命令搞定。  

* **安装**  
上传完成之后，在 hadoop 这个用户目录创建一个tools文件夹来专门管理文件（/home/hadoop/tools/），使用 命令``tar -zxvf``将JDK解压到tools文件夹中:
```
tar -zxvf jdk-7u65-linux-i586.tar.gz -C tools/
```
解压完成，进入jdk1.7.0_65/bin看到我们熟悉的java命令则安装成功
* **配置java的环境变量**  
为了方便我们在系统中使用java命令，需要将/bin目录添加到环境变量中，使用命令：  
``vi /etc/profile``  
在最后一行加入：  
```shell
export JAVA_HOME=/home/hadoop/tools/jdk1.7.0_65
export PATH=$PATH:$JAVA_HOME/bin
```
保存之后，刷新配置让其更改生效：``source /etc/profile``  
在/bin文件夹外面使用``java -version``能看到版本信息  
```bash
java version "1.7.0_65"
Java(TM) SE Runtime Environment (build 1.7.0_65-b17)
Java HotSpot(TM) Client VM (build 24.65-b04, mixed mode)
```
到此为止，Java已经安装完成。  

## 二、安装Hadoop  
Linux不一定要用的很深，但是要用熟
使用上传JDK的方式将hadoop上传到Linux中  
安装方式与安装jdk同理，解压到tools文件夹中即可：  
``tar -zxvf hadoop-2.9.0.tar.gz -C tools/``  
* **配置hadoop的环境变量**  
在 JAVA_HOME 之后加入 hadoop的 /bin 和 /sbin  
``vi /etc/profile``
```shell
export JAVA_HOME=/home/hadoop/tools/jdk1.7.0_65
export HADOOP_HOME=/home/hadoop/tools/hadoop-2.9.0
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
* **修改hadoop配置文件**  
解压完成后开始修改hadoop的配置文件  
配置文件位于``/hadoop-x.x.x/etc/hadoop``目录下  
更改如下5个配置文件：  


1.``vi hadoop-env.sh``  
hadoop 环境变量脚本文件 hadoop-env.sh
```shell
export JAVA_HOME=/home/hadoop/tools/jdk1.7.0_65  
```
注意这里一定要是java的绝对路径，不可以用$JAVA_HOME代替。  

2.``vi core-site.xml``  
hadoop 核心配置文件 core-site.xml  
```shell
<configuration>
  <property>
      <name>fs.defaultFS</name>
      <value>hdfs://master:9000</value>         
    </property>  
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/tmp</value>
  </property>
</configuration>
```
这里的<value>hdfs://master:9000</value> ：master表示主机名，默认是localhost（也可以是IP地址），默认端口是9000。/home/hadoop/tmp为hadooop运行临时文件的目录，自己合理选择文件夹。  

3.``vi hdfs-site.xml``  
hdfs 配置文件 hdfs-site.xml
```shell
<configuration>
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>
</configuration>
```
指定HDFS副本的数量  

4.``vi mapred-site.xml``  
MapReduce 配置文件 mapred-site.xml
```shell
<configuration>
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
</configuration>
```
5.``vi yarn-site.xml``  
yarn配置文件yarn-site.xml  
```shell
<configuration>
<property>                                                       
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
</configuration>
```  
指定哪些机器上面要启动namenode，我们这里只有一台机器，先加入本机的主机名,如果有多台机器,在后面依次添加主机名  
最后修改slaves  ``vi slaves``  
```shell
[hadoop@master hadoop]$ vi slaves
master
```

## 三、启动Hadoop  

1.格式化HDFS文件系统，对NameNode进行初始化  
``hdfs namenode -format``  

2.启动HDFS (启动NameNode和DataNode守护进程)  
``sbin/start-dfs.sh``  

3.启动YARN (启动ResourceManager 和 NodeManager 守护进程)  
``sbin/start-yarn.sh``  

4.验证是否启动成功，查看进程命令： ``jps``  
如看到以下进程，则表示hadoop正常启动  
```shell
3039 ResourceManager
2745 DataNode
2885 SecondaryNameNode
3311 NodeManager
2634 NameNode
3351 Jps
```  

启动成功后，可以在浏览器中输入`` http://主机地址:50070``访问web界面，查看 NameNode 和 Datanode 信息，还可以在线查看 HDFS 中的文件。我这里通过主机名 master 访问(需要修改windows中hosts文件主机名对应的ip)，或者通过主机的ip地址也可以访问。  

![](http://wx3.sinaimg.cn/large/005TBZ5oly1fnndaj82y2j30xe0iy755.jpg)  

界面查看任务的运行情况：``http://主机地址:8088/cluster``  

![](http://wx3.sinaimg.cn/large/005TBZ5oly1fnndrg8y6rj311s0cswfx.jpg)  

至此，Hadoop伪分布式已成功搭建
