---
title: kafka源码解析之 Producer
date: 2016-10-09 19:16:39
tags: kafka
categories: 技术
---
国庆长假，偷空将 Kafka producer 部分基本看完了。来来回回看了好几遍，但有些细节还是没有搞太明白，不过大概流程算是梳理清楚了。这篇主要将主线梳理清楚，之后还会继续研读代码，争取尽善尽美。

<!--more-->

### 主线

主线主要分为两个部分，分别介绍之：

#### 初始化

![image](http://o8ahjnaal.bkt.clouddn.com/kafka%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B.png)

初始化框架流程大概如上图所示。

##### 初始化配置参数：

因为 Producer 要发送消息到 broker，所以首先要讲消息序列化为适合网络传输的对象，故需要序列化类，其次，要决定发往哪个分区，故需要分区类，然后 kafka 为了实现一些度量的功能，比如对于效率的测试等等，所以也许要初始化一些度量类，最后就是一些其他与发送消息相关的参数，如每次请求量的大小、发送请求的内存大小，发送失败需要重试的时延等等。

##### 初始化核心类：

从图中可以看出来，核心类主要有四个：

1.Metadata 

这里面存放了这个 producer 发送的 topic ，version 信息（用于维护发送机制），和 cluster（Kafka broker 集群的抽象，里面存放了所有的可用节点的集合，和这些节点与partition的映射关系）。

2.ReccordAccumulator

kafka 将每个要发送的消息添加到 RecordBatch 中（这个机制在下面发送消息部分详细介绍）这个类的主要功能就是将要发送的消息添加到Deque队列里面的 RecordBatch 里面，同时如果 RecordBatch 满了，则继续新建一个 RecordBatch，再将其入队。

3.NetworkClient

这个类是 KafkaClient 接口的一个实现，用来执行实际的 IO 请求响应，是面向用户（Producer 和 Consumer）的最外层的一个 Client。

4.Sender 

这是一个后台线程类，主要用来发送 Producer 的request 到 broker。

5.度量类

#### 发送消息

kafka 发送消息的大概示意图如下：

![image](http://o8ahjnaal.bkt.clouddn.com/kafka%20%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)

调用 Kafka 的 producer 的 send 方法发送消息，实际上是将消息添加到 RecordBatch 里面。RecordBatch 是一个 deque 的节点。

可以看到 Kafka 的消息发送机制，并不是来一条消息发送一条消息，而是为了提高效率，将消息先放到 RecordBatch 的数据结构中，如果 RecordBatch 已满，则再新建一个 RecordBatch ，将消息放进去，同时唤醒 Sender 后台线程去从这个 Deque 里面去取消息（当然在初始化 Kafka Producer 的时候，已经同时初始化了 Sender 线程，此线程会一直循环去 Deque 中去取已经满了的 RecordBatch ，之后通过 NetworkClient 发送到 Broker 中 ）


### 总结

这一篇，没有贴代码，主要用两个图把 Kafka 的机制大概介绍了下，从最外层的轮廓中去看，有一个初步的印象，否则会徜徉在代码的大海里，不可自拔。

在后面的文章中，会尽量将每个细节都抠出来，对照代码细讲。
























