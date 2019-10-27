---
title: 在mac上搭建单机版hadoop环境
date: 2018-03-11 22:47:29
tags: hadoop 集群
categories: 技术
---
初入大数据门槛，废话少说，从搭建单机版hadoop开始。

<!--more-->

## 下载离线 hadoop 安装包

从网上搜很多博客，提到的都是 brew 一键安装，但是我亲自用过，brew 过程非常缓慢，并不如自己下载离线安装包来的快。

hadoop 下载地址：https://archive.apache.org/dist/hadoop/common/hadoop-2.7.3/

下载名为 hadoop-2.7.3.tar.gz 的安装包，之后解压即可。


## 修改 hadoop 配置文件

执行命令 

cd /etc/hadoop

vim hadoop-env.sh

export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_111.jdk/Contents/Home

vim core-site.xml

这个配置文件用于指明namenode 的主机名和端口。

	<configuration>
		<property>
	    	<name>fs.defaultFS</name>
	    	<value>hdfs://0.0.0.0:9000</value>
		</property>
		<property>
	    	<name>hadoop.tmp.dir</name>
	    	<value>/Users/lixuehan/hadoop-2.7.3/temp</value>
		</property>
	</configuration>

将这部分直接拷贝进去即可。

修改 hadfs-site.xml 
这个配置文件用来指定副本数，fs.namenode.name.dir指明fsimage存放目录，多个目录用逗号隔开。dfs.datanode.data.dir指定块文件存放目录，多个目录逗号隔开。



		<configuration>
			<property>
	    		<name>dfs.replication</name>
	    		<value>1</value>
			</property>  
			<property>
	    		<name>dfs.namenode.name.dir</name>
	    		<value>file:/Library/hadoop-2.7.3/tmp/hdfs/name</value>
			</property>
			<property>
	    		<name>dfs.datanode.data.dir</name>
	    		<value>file:/Library/hadoop-2.7.3/tmp/hdfs/data</value>
			</property>
			<property>
	    		<name>dfs.webhdfs.enabled</name>
	    		<value>true</value>
			</property>
			<property>
	    		<name>dfs.http.address</name>
	    		<value>0.0.0.0:50070</value>
			</property>
		</configuration>





修改 mapred-site.xml

	<configuration>
	
	<property>
	    <name>mapreduce.framework.name</name>
	    <value>yarn</value>
	</property>
	
	</configuration>

修改yarn-site.xml

	<configuration>
	<!-- Site specific YARN configuration properties -->
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
	
	</configuration>


​    
### 配置hadoop环境变量

vim ~/.bash_profile

export HADOOP_HOME=/Users/eleme/Documents/ProgramFiles/apache-software-foundation/hadoop-2.7.3/hadoop [此处写hadoop 解压后的目录]

export PATH=$PATH:$HADOOP_HOME/bin

source /etc/profile

### 启动 hadoop

	//进入hadoop安装目录
	cd $HADOOP_HOME
	//初始化namenode
	hdfs namenode -format
	//启动hdfs
	sbin/start-dfs.sh 
	//启动yarn
	sbin/start-yarn.sh

启动完hdfs访问: http://localhost:50070

启动完yarn访问: http://localhost:8088

#### 参考链接
https://www.cnblogs.com/landed/p/6831758.html



