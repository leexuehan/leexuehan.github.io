---
title: ftp协议学习
date: 2016-06-13 23:17:23
tags: ftp
categories: 技术
---
## 什么是ftp？

ftp,即文件传输协议（File Transfer Protocol）。通俗理解就是从服务器上面下载文件或者上传文件的一种协议。和http一样都属于应用层协议，但是ftp比较http协议更为复杂。

<!--more-->

他们的底层都建立在TCP/IP传输协议上，但是ftp较为复杂的一点是：它建立了两个TCP链路，一个用来传输命令叫命令链路，另一个用来传输数据，叫数据链路。如下图所示：

![](http://o8ahjnaal.bkt.clouddn.com/ftp%E9%93%BE%E8%B7%AF.png)


## 主动模式和被动模式

ftp传输方式主要有两种方式：主动模式和被动模式。

下面分别用两张图来表示这两种模式中，两条TCP链路的建立过程：

### 主动模式

![](http://o8ahjnaal.bkt.clouddn.com/ftp%E4%B8%BB%E5%8A%A8%E6%A8%A1%E5%BC%8F.jpg)

第1、2步就是建立命令链路的过程,由客户端负责建立，3、4步是建立数据链路的过程，由服务端负责建立。可以看到所谓主动模式就是服务端主动把请求告知客户端某个端口：我已经开了哪些端口，你来连接我吧。客户端链接的一般都是server端的20端口。


### 被动模式

![](http://o8ahjnaal.bkt.clouddn.com/ftp%E8%A2%AB%E5%8A%A8%E6%A8%A1%E5%BC%8F.jpg)

从图中可以看出来，第1、2步是建立命令链路的过程，3、4步是建立数据链路的过程，两个链路的建立都是由客户端负责建立。可以看到所谓的被动模式就是客户端发送请求到服务端的某个端口，告诉服务端：我已经开放了那些端口，你来链接我吧。而且客户端链接的一般都是server端的高位端口。




总结以上两种模式就是：在建立命令链路的过程基本相同，建立数据链路的不同，“主动”或者“被动”都是针对服务端而言的，所谓主动模式就是：主动告知对方自己打开了某端口，而被动模式就是：被对方首先告知自己开放了某端口。他们之间建立命令链路的过程是相同的，在建立数据链路的过程中有所区别。

而且主动模式和被动模式还有另外一个区别就是：server数据链路的端口不一样，前者是20端口，后者是高位端口。

如果你还没有理解，可以参看下面这段在 stackoverflow 里面的高票回答：

>in an active mode configuration, the server will attempt to connect to a random client-side port. So chances are, that port wouldn't be one of those predefined ports. As a result, an attempt to connect to it will be blocked by the firewall and no connection will be established.

>A passive configuration will not have this problem since the client will be the one initiating the connection. Of course, it's possible for the server side to have a firewall too. However, since the server is expected to receive a greater number of connection requests compared to a client, then it would be but logical for the server admin to adapt to the situation and open up a selection of ports to satisfy passive mode configurations.

>So it would be best for you to configure server to support passive mode FTP. However, passive mode would make your system vulnerable to attacks because clients are supposed to connect to random server ports. Thus, to support this mode, not only should your server have to have multiple ports available, your firewall should also allow connections to all those ports to pass through!

>To mitigate the risks, a good solution would be to specify a range of ports on your server and then to allow only that range of ports on your firewall.

大意就是：

主动模式因为是服务端去尝试连接客户端的一个随机的端口，所以很有可能这个随机端口不是预定义好的端口，结果是服务端连接客户端的请求被拒之防火墙外，数据连接建立失败；

而被动模式则有没有这个问题，因为客户端负责初始化两个连接（命令链路和数据链路），当然也有可能在服务端一侧也有防火墙，但是，由于它是服务器，要接受比客户端多得多的请求，所以服务端一般会打开一系列的端口，去满足被动模式的链接。

所以，**最好把服务器配置为被动模式**。

但是这样也会有潜在的风险，这样会使你的系统更易受到攻击，因为客户端要链接服务端的一个随机的端口，所以为了支持这种模式，不仅需要你的服务端的这些端口开放，同时也必须是你的防火墙允许这些请求经过。所以为了减轻风险，最好将你的服务器开放的端口定在一个合适的范围，只有这个范围内的端口才可以开放链接。

