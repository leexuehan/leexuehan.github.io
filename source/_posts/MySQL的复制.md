---
title: MySQL的复制
tags:
  - MySQL
date: 2015-08-26 10:45:44
categories: 技术
---

### MySQL库的复制原理和流程

关于MySQL 的复制主要涉及三个线程：主库线程，I/O线程和SQL线程。可以从下图看出来：

![](http://i.imgur.com/qCAFwnJ.png)

1.主库线程把自己的数据更改记录到 Binnary log 中

2.从库的I/O线程向主库发起读取 BinLog 的请求

3.主库将 BinLog 推送到从库

4.I/O线程将日志推送到自己的 Relay Log（中继日志）中

5.SQL线程从中继日志中读取SQl并记入从库的 BinLog 中，然后 Flush 到硬盘。

