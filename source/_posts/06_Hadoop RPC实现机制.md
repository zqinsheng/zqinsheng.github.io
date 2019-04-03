---
title: Hadoop RPC实现机制
date: 2018-2-2 00:22:34
tags:
- hadoop
categories:
- hadoop
---

## 一、RPC基础概念

RPC 是Remote Procedure Call（远程过程调用）的简写，是一种通过网络从远程计算机上请求服务的机制，封装了具体实现，使用户不需要了解底层网络技术。  
RPC采用客户机/服务器模式,主要应用于一些分布式的系统，客户机上的一个请求程序,调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息的到达为止。当一个调用信息到达，服务器获得进程参数，根据参数作出答复，发送答复信息，然后等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。用简单的话说，就是一台计算机程序远程调用另外一台计算机的程序。
RPC对于我们来说是透明的，在调用的过程中感觉不到其间涉及跨机器间的通信，而是感觉像是在执行一个本地调用。我们不用去关心底层的网络通信细节，这是RPC中的访问透明性。  

![](http://wx3.sinaimg.cn/large/005TBZ5oly1fo8cmi1hldj30kx0beq4i.jpg)

1. 通信模块：客户端与服务的的请求应答消息，TCP/IP的socket机制
2. stub程序：代理程序，为了保证RPC的透明性，它在客户端表现的就像调用本地程序一样，客户端发送请求参数之后，服务器端stub程序会解码对应结果、调用相应的服务过程返回结果给服务端。
3. 客户端/服务端过程：即请求的发出者与请求的处理者
4. 调度程序：调度程序接受来自通信模块的请求消息，并根据其中的标识选择一个Stub程序进行处理。

<!-- more -->
## 二、Hadoop中RPC的Demo

hadoop 的 common 包中包含有 RPC 的包，有了这个 RPC 的包才能进行 RPC 操作，通过一个demo程序来感受一下RPC。  
模拟用户登陆成功：从客户端调用远程服务端的登陆方法，登陆成功并返回登陆结果。

|          | 客户端        | 服务端                        |
| -------- | ------------- | ----------------------------- |
| 环境     | win10/eclipse | centerOS/eclipse              |
| 网络配置 | 暂时用不到        | 主机名：master(192.168.2.100) |
| 项目详情         |     ![](http://wx1.sinaimg.cn/large/005TBZ5oly1fobqv7gk48j306r043a9y.jpg)          |                 ![](http://wx4.sinaimg.cn/large/005TBZ5oly1fobqq0xd7bj308w05cweg.jpg)              |

### 服务端  

** 1.定义一个LoginService协议接口 **  
RPC通信是通过 client 和 server 之间的接口通信，定义 server 对外服务的接口：

```java
public interface LoginService {
  //定义协议版本号，通过这个版本号来确认client和server之间的通信，不同版本号之间不能相互通信
  public static final long versionID = 1L;

  public String login(String username);
}
```

** 2.定义LoginService的实现：LoginServiceImpl **

```java
public class LoginServiceImpl implements LoginService {

	public String login(String username) {
		return username+" is login !";
	}

}
```

** 3.构造并启动RPC server **

```java
public class Start {

	public static void main(String[] args) throws HadoopIllegalArgumentException, IOException {
    //创建RPC.Builder实例，设置一些必要的参数
		Builder builder = new RPC.Builder(new Configuration());
    //设置主机名
		builder.setBindAddress("master");
    //设置端口号
    builder.setPort(6666);
    //协议接口
    builder.setProtocol(LoginService.class);
    //业务逻辑实例
    builder.setInstance(new LoginServiceImpl());
		//构造RPC Server实例
		Server server = builder.build();
    //启动服务，处于监听状态
    server.start();
	}
}
```

### 客户端
** 1.将服务端的 LoginService 协议接口拷贝到客户端中 **

![](http://wx1.sinaimg.cn/large/005TBZ5oly1fobqv7gk48j306r043a9y.jpg)

注意客户端中的 LoginService 同样需要定义协议版本号并与服务端保持一致

```java
public interface LoginService {

  public static final long versionID = 1L;

  public String login(String username);
}
```

** 2.调用服务端的login方法 **

客户端代码的核心在于RPC.getProxy()，它可以返回一个代理对象，这个代码对象就是服务器端对象的代理

```java
public class LoginController {
	public static void main(String[] args) throws IOException {
    /**
      通过RPC拿到代理对象
      RPC.getProxy(协议接口, 版本号, 通信地址:主机名，端口号 , 配置信息);
    */
		LoginService proxy = RPC.getProxy(LoginService.class, 1L, new InetSocketAddress("master", 6666), new Configuration());
    //调用login方法,就像调用本地方法一样
		String loginUser = proxy.login("JasonZhou");
		System.out.println(loginUser);
    //关闭连接
    RPC.stopProxy(proxy);
	}
}
```

运行成功，在控制台看到输出信息，调用远程方法成功 ！
``JasonZhou is login !``

整个过程如下图所示
![](http://wx1.sinaimg.cn/large/005TBZ5oly1fobst8xn04j31kw0qgqv5.jpg)
