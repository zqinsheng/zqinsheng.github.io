---
title: ssh免密登陆
date: 2018-1-10 00:05:28
tags:
- hadoop
categories:
- hadoop
---

在启动和停止hadoop的时候，我们发现需要输入很多次密码，在伪分布式的环境中，就一个节点，一个 NameNode 和几个DataNode，输入几次密码还可以接受，但是如果有1000个节点，这样的操作要重复1000次，这是我们不能接受的。由此我们要使用一种叫做 SSH 的登陆方式来帮助我们免密登陆。  

## 一、什么是SSH
SSH(Secure Shell)是一项创建在应用层和传输层基础上的安全协议，早期互联网通信都是明文通信，一旦被截获将暴露内容。1995年，芬兰研究员Tatu Ylonen设计了SSH协议，将登录信息全部加密，成为互联网安全的一个基本解决方案，迅速在全世界获得推广，目前已经成为Linux系统的标准配置。  

如果一个用户从本地计算机，使用SSH协议登录另一台远程计算机，我们就可以认为，这种登录是安全的，即使被中途截获，密码也不会泄露。  
简单的说，SSH就是一种网络协议，用于计算机之间的加密登录。  
<!-- more -->
## 二、hdoop中的SSH免密登陆  
### 基本原理  
有A和B两台机器
1. 在A上创建一对密钥对，包括一个公钥和一个私钥 （公钥文件：~/.ssh/id_rsa.pub； 私钥文件：~/.ssh/id_rsa）。
2. 把A公钥放到B的（~/.ssh/authorized_keys）文件中, 自己保留好私钥.
3. 在使用ssh登录时,ssh程序会发送A私钥去和B上的A公钥做匹配.如果匹配成功就可以登录了。  

假设我们有两台机器，A实现SSH免密登陆访问B  
**A.hadoop@master(192.168.2.100)**  
**B.hadoop@slave(192.168.2.128)**  

s
**第一步：生成密钥对** ``ssh-keygen -t rsa``  
在A机器上生成SSH密钥对，rsa 是加密算法，询问其密码直接回车默认为空，询问保存路径时直接回车采用默认路径
```
[hadoop@master /]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/hadoop/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/hadoop/.ssh/id_rsa.
Your public key has been saved in /home/hadoop/.ssh/id_rsa.pub.
The key fingerprint is:
2b:42:8e:3d:ab:6e:98:e4:67:81:6c:5b:3d:86:ba:a8 hadoop@master
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|                 |
|. . +   S        |
| = O +   .       |
|+o= B o .        |
|++.o + .         |
|E+*..            |
+-----------------+
```  
一路回车，此时生成的密钥对：id_rsa和id_rsa.pub，会默认存储在"/home/hadoop/.ssh"目录下。    
```shell
/home/hadoop/.ssh
[hadoop@master .ssh]$ ll
total 12
-rw-------. 1 hadoop hadoop 1675 Jan 28 22:35 id_rsa
-rw-r--r--. 1 hadoop hadoop  395 Jan 28 22:35 id_rsa.pub
-rw-r--r--. 1 hadoop hadoop 1188 Jan 28 22:15 known_hosts
[hadoop@master .ssh]$
```  
**第二步：将生成的A公钥远程拷贝到B机器  hadoop@slave(192.168.2.128)**  
拷贝命令：``scp id_rsa.pub slave:/home/hadoop``  

**第三步：将公钥添加到授权文件"authorized_keys"**  
登陆到B机器，在B机器上将A公钥追加到授权文件中  
``cat id_rsa.pub >> .ssh/authorized_keys``  

**第四步：修改文件"authorized_keys"权限**  
这一步很重要，否则SSH登陆时仍然需要密码  
``chmod 700 ~/.ssh``  
``chmod 600 authorized_keys``  

**第五步：修改SSH配置文件**  
``vi /etc/ssh/sshd_config``  

找到如下三条信息，默认是被注释的，将注释去掉，这三条信息的含义分别是：  
RSAAuthentication yes # 启用 RSA 认证  
PubkeyAuthentication yes # 启用公钥私钥配对认证方式  
AuthorizedKeysFile .ssh/authorized_keys # 公钥文件路径（和authorized_keys文件路径相同）  

```shell
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile     .ssh/authorized_keys
#AuthorizedKeysCommand none
#AuthorizedKeysCommandRunAs nobody
```  

**第六步：重启SSH服务**  
```shell
[root@slave .ssh]# service sshd restart
Stopping sshd: [  OK  ]
Starting sshd: [  OK  ]
```
**第七步：验证，在 A机器master上使用 SSH登录 B机器 slave**  

```Shell
[hadoop@master /]$ ssh slave
Last login: Sun Jan 28 08:33:08 2018 from 192.168.2.100
[hadoop@slave ~]$
```  
到此就OK了，或者第一次需要输入密码，以后再次登陆就不需要输入密码了。如果配置完后每次都要输入密码，大多是因为文件权限的问题，重新检查.ssh文件夹和authorized_keys的访问权限。  

## 三、伪分布式集群启动免密  
回到一开始的问题，在启动hadoop集群的时候，我们需要输入很多次密码，用SSH的方式免密登陆，同样的道理，将自己的公钥添加到自己机器的授权文件中即可。  
``[hadoop@master .ssh]$ cat id_rsa.pub >> ./authorized_keys``  
修改权限：  
``[hadoop@master .ssh]$ chmod 600 authorized_keys``  

启动和停止一下hadoop看看，是不是就不用输入密码了。
