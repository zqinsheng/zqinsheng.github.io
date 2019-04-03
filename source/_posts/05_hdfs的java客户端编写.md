---
title: HDFS的JAVA客户端编写
date: 2018-1-27 01:47:12
tags:
- hadoop
categories:
- hadoop
---

之前一直使用 HDFS Shell 命令来完成对 HDFS 的操作，当然也可以使用 Java 来对 HDFS 实现操作。
接下来通过一些例子来看一下 JAVA 是如何对 HDFS 实现操作的。  

### 在eclipse中新建一个java项目   
将Linux中hadoop安装目录下，配置好的``core-site.xml`` ``hdfs-site.xml``，mapred-site.xml，yarn-site.xml都拷贝到eclipse工作空间
![](http://wx2.sinaimg.cn/large/005TBZ5oly1fo4ufraku9j309c05t0sq.jpg)  

要对hdfs进行操作，需要两个核心jar包，这两个jar包可以在hadoop安装包中找到（hadoop-x.x.x\share\hadoop），导入这两个文件夹中对应的jar包以及他们的依赖包(在这两个目录下面的lib文件夹中)
![](http://wx4.sinaimg.cn/large/005TBZ5oly1fo9isn3lvqj304m01b0s3.jpg)  
将这两个jar包导入到项目中
![](http://wx1.sinaimg.cn/large/005TBZ5oly1fo4uamux7xj30r90fn75r.jpg)
<!-- more -->
或通过maven，添加相应版本的依赖

pom.xml中加入这两个依赖

```xml
		<!-- hadoop hdfs类库 -->
		<dependency>
			<groupId>org.apache.hadoop</groupId>
			<artifactId>hadoop-hdfs</artifactId>
			<version>2.x.x</version>
		</dependency>
		<!-- hadoop 公共类库 -->
		<dependency>
			<groupId>org.apache.hadoop</groupId>
			<artifactId>hadoop-common</artifactId>
			<version>2.x.x</version>
		</dependency>
```  

### 访问HDFS的两种方式  
访问 HDFS 的核心类是**FileSystem**，**FileSystem** 是文件系统的抽象，HDFS是分布式文件系统对 **FileSystem** 的实现，**FileSystem** 有很多文件系统的实现，不论底层文件系统的具体实现是什么样的，文件系统 **FileSystem** 统一提供了访问接口，直接通过 **FileSystem** 来进行操作，如此即可解耦合。
如下是 **FileSystem** 的继承关系：  

![](http://wx3.sinaimg.cn/large/005TBZ5oly1fo8d68895jj30jj0e9t9k.jpg)

**通过FileSystem访问HDFS**
1. 设置默认配置文件

```java
/*
方式1：
默认读取classpath下的xxx.site.xml配置文件，并解析其内容，封装到conf对象中。
设置默认文件系统、设置run Configuration的参数: -DHADOOP_USER_NAME=hadoop  
*/   
FileSystem fs = null;
Configuration conf = new Configuration();      
fs = FileSystem.get(conf);   
```  

2. 在构造函数中设置配置信息  

```java
/*
方式2:在此方法的参数中设置默认文件系统、用户名
根据配置信息，去获取一个具体文件系统的客户端操作实例对象
会覆盖掉配置文件中读取的值
*/  
FileSystem fs = null;
Configuration conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://master:9000/");
fs = FileSystem.get(new URI("hdfs://master:9000/"), conf, "hadoop");
```  
### FileSystem操作实例
项目新建完成，先来测试从hdfs上下载文件，从hdfs中将 jdk-7u65-linux-i586.tar.gz 下载到本地

```java
package org.zqinsheng.hdfs;

import java.io.FileOutputStream;
import java.io.IOException;

import org.apache.commons.io.IOUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class HdfsUtil {
	public static void main(String[] args) throws IOException {
		//hdfs的配置信息,上述的两种配置方式,这里采用第一种
		Configuration conf = new Configuration();
		//根据配置信息，去获取一个具体文件系统的客户端操作实例
		FileSystem fs = FileSystem.get(conf);
		//hdfs文件存放的路径
		Path path = new Path("hdfs://master:9000/jdk-7u65-linux-i586.tar.gz");
		//通过一个输入流获取将要下载的文件
		FSDataInputStream is = fs.open(path);
		//通过一个输出流指定下载到哪
		FileOutputStream out = new FileOutputStream("h:\\download.jar");
		//commons包中提供的工具类,将一个输入流拷贝到输出流中
		IOUtils.copy(is, out);
	}
}
```

也许会出现这样的错误，原因是java不认识hdfs://这样的路径，本地文件系统不知道这是什么路径，一般都是**配置文件**的问题，请检查是否将 ``core-site.xml、hdfs-site.xml``文件拷贝到src目录下，或是检查配置参数。
![](http://wx2.sinaimg.cn/large/005TBZ5oly1fo4swh96k2j30zq07xq3w.jpg)   

或者是这样的错误，权限被拒绝，原因还是在windows中我们的用户不是hdfs所指定的用户，如果在Linux中使用eclipse则不会出现这样的情况。

![](http://wx2.sinaimg.cn/large/005TBZ5oly1fo5xyyhyvgj30zf0fnjt1.jpg)

解决办法：
* 可以在run as->run
configuration->arguments里面添加-DHADOOP_USER_NAME=用户名  

* 或者是通过FileSystem.get()进行user的指定，fs = FileSystem.get(new URI("hdfs://master:9000/"), conf, "hadoop");



上面的下载文件是一种相对底层的写法，**FileSystem** 已经帮我们将这些操作封装好了，我们直接调用这些方法即可完成相对应的操作。通过一个**Unit**测试代码来看一下封装好的方法。  
```java
package org.zqinsheng.hdfs;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.LocatedFileStatus;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.fs.RemoteIterator;
import org.junit.Before;
import org.junit.Test;

public class TestHdfs {

	FileSystem fs = null;

	// 每次执行方法前都获取FileSystem实例对象
	@Before
	public void init() throws IOException, Exception, Exception {
		Configuration conf = new Configuration();
		fs = FileSystem.get(new URI("hdfs://master:9000"), conf, "hadoop");
	}

	/**
	 * 上传文件
	 *
	 */
	@Test
	public void upload() throws Exception, IOException {
		fs.copyFromLocalFile(new Path("f://HelloWorld.txt"), new Path("hdfs://master:9000/testHdfs"));
	}

	/**
	 * 下载文件
	 *
	 * 如果报空指针异常
	 *
	 * 解决方法:将fs.copyToLocalFile(hdfsPath,localPath);
	 *
	 * 改为fs.copyToLocalFile(false,hdfsPath,localPath,true);
	 */
	@Test
	public void download() throws Exception, IOException {
		fs.copyToLocalFile(new Path("hdfs://master:9000/testHdfs/HelloWorld.txt"), new Path("h:/HelloWorld.txt"));
	}


	/**
	 * 创建文件夹
	 * @throws IOException
	 * @throws IllegalArgumentException
	 */
	@Test
	public void mkdir() throws IllegalArgumentException, IOException {
		fs.mkdirs(new Path("hdfs://master:9000/aa/bb/cc"));
	}


	/**
	 * 删除文件或文件夹
	 * @throws IOException
	 * @throws IllegalArgumentException
	 */
	@Test
	public void delete() throws IllegalArgumentException, IOException {
		fs.delete(new Path("hdfs://master:9000/aa"),true);
	}


	/**
	 * 查询指定目录下的文件，不包括文件夹，包括子文件夹下的文件
	 * @throws IOException
	 * @throws IllegalArgumentException
	 * @throws FileNotFoundException
	 */
	@Test
	public void listFile() throws FileNotFoundException, IllegalArgumentException, IOException {
		//获取到文件状态的迭代器,true表示递归遍历
		RemoteIterator<LocatedFileStatus> files = fs.listFiles(new Path("hdfs://master:9000/"),true);
		//遍历迭代器
		while(files.hasNext()) {
			//得到每个文件状态
			LocatedFileStatus file = files.next();
			//拿到LocatedFileStatus对象后，可以通过这个对象获取文件的名称、路径、块大小、权限、所属者等等信息
			String name = file.getPath().getName();
			System.out.println(name);
		}
	}

	/**
	 * 查询指定目录下的文件和文件夹
	 * @throws Exception
	 * @throws IllegalArgumentException
	 * @throws FileNotFoundException
	 */
	@Test
	public void listFileAndDir() throws FileNotFoundException, IllegalArgumentException, Exception {
		//列出文件或文件夹信息，不提供递归遍历
		//得到文件的状态
		FileStatus[] listStatus = fs.listStatus(new Path("hdfs://master:9000/"));
		for(FileStatus fileStatus:listStatus) {
			//获取文件名称
			String name = fileStatus.getPath().getName();
			System.out.println(name);
		}
	}
}

```
