---
title: Hadoop集群搭建教程
date: 2020-09-17 22:52:16
tags: Hadoop
categories: 其它
---
### 一、环境安装

#### 1. 准备虚拟机

使用VMware或hyper-v创建三台虚拟机（系统为Ubuntu18.04）。

#### 2. 设置静态IP、修改主机名和hosts

- 设置静态IP

  修改目录/etc/netplan下的配置文件01-netcfg.yaml（文件名可能不一样）

  ```shell
  root@Hadoop:~# vim /etc/netplan/01-netcfg.yaml
  ```

  将dhcp4更改为no，并设置IP地址、网关和DNS服务器，配置如下

  ```shell
  # This file describes the network interfaces available on your system
  # For more information, see netplan(5).
  network:
    version: 2
    renderer: networkd
    ethernets:
      eth0:
        dhcp4: no
        addresses: [172.22.160.101/20]    #IP地址和掩码
        gateway4: 172.22.160.1            #网关
        nameservers:                      #DNS
                addresses: [114.114.114.114]
  
  ```

  另外两台主机的IP分别为：172.22.160.102、172.22.160.103

  **注意：**IP设置成自己的，确保三台主机在同一局域网下，且能够相互ping通

  

- 修改主机名

  将3台主机名修改成不同的名称，分别为Hadoop101、Hadoop102、Hadoop103，便于区分。

  ```shell
  vi /etc/hostname
  ```

  

- 修改hosts文件

  ```shell
  vi /etc/hosts
  ```

  添加以下三条记录（IP需要修改成自己的）

  ```
  172.22.160.101	Hadoop101
  172.22.160.102	Hadoop102
  172.22.160.103	Hadoop103
  ```

#### 3. 配置ssh

##### 1. 安装ssh

检查是否安装了ssh

```shell
root@Hadoop101:~# ps -e | grep ssh
 1048 ?        00:00:00 sshd
 1097 ?        00:00:00 sshd
 1217 ?        00:00:00 sshd
```

若没有出现上述信息，则需要安装ssh

```shell
apt-get install openssh-server
```

##### 2. 允许root通过ssh登录

修改sshd_config文件

```shell
hadoop@Hadoop102:~$ sudo vim /etc/ssh/sshd_config
```

找到PermitRootLogin，修改为如下

```shell
#LoginGraceTime 2m
#PermitRootLogin prohibit-password
PermitRootLogin yes
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

##### 3. 设置免密登录

在主机Hadoop101上生成密钥公钥对

```shell
root@Hadoop101:~# ssh-keygen -t rsa
```

将公钥发送到主机Hadoop102和Hadoop103

```shell
root@Hadoop101:~# ssh-copy-id hadoop102
root@Hadoop101:~# ssh-copy-id hadoop103
```

#### 4. 编写分发脚本

一般情况下，服务器的数量比较多，在每台服务器上分别配置Hadoop不太现实，rsync可以将文件同步到其他服务器，以下脚本可以群发

```shell
#!/bin/bash
#1 获取输入参数个数，如果没有参数，直接退出
pcount=$#
if((pcount==0)); then
echo no args;
exit;
fi

#2 获取文件名称
p1=$1
fname=`basename $p1`
echo fname=$fname

#3 获取上级目录到绝对路径
pdir=`cd -P $(dirname $p1); pwd`
echo pdir=$pdir

#4 获取当前用户名称
user=`whoami`

#5 循环，这里host根据自己的节点数和主机名设置
for((host=102; host<104; host++)); do
        #echo $pdir/$fname $user@hadoop$host:$pdir
        echo --------------- hadoop$host ----------------
        rsync -rvl $pdir/$fname $user@hadoop$host:$pdir
done
```

将脚本放在/usr/sbin目录下，并赋予777权限

#### 5. 安装jre和Hadoop

解压server-jre-8u261-linux-x64.tar.gz到/usr/java

```shell
root@Hadoop101:~# mkdir /usr/java
root@Hadoop101:~# tar -zxvf server-jre-8u261-linux-x64.tar.gz
root@Hadoop101:~# mv jdk1.8.0_261/ /usr/java/
```



解压hadoop-3.2.1.tar.gz到/usr/hadoop

```shell
root@Hadoop101:~# mkdir /usr/hadoop
root@Hadoop101:~# tar -zxvf hadoop-3.2.1.tar.gz
root@Hadoop101:~# mv hadoop-3.2.1 /usr/hadoop/
```

添加环境变量

```shell
vim /etc/profile
```

添加以下内容：

```shell
export JAVA_HOME=/usr/java/jdk1.8.0_261
export HADOOP_HOME=/usr/hadoop/hadoop-3.2.1
export PATH=$PATH:${JAVA_HOME}/bin:${HADOOP_HOME}/bin
```

### 二、配置Hadoop集群

切换到目录/usr/hadoop/hadoop-3.2.1/etc/hadoop/，修改配置文件

#### 1. hadoop-env.sh

在该文件中添加JAVA_HOME

```shell
# The java implementation to use. By default, this environment
# variable is REQUIRED on ALL platforms except OS X!
export JAVA_HOME=/usr/java/jdk1.8.0_261
```

#### 2. core-site.xml

```xml
<!-- 指定HDFS中NameNode的地址 -->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop101:9000</value>
</property>

<!-- 指定Hadoop运行时产生文件的存储目录 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/hadoop/hadoop-3.2.1/tmp</value>
</property>

```

#### 3. hdfs-site.xml

```xml
<!-- hdfs存储数据的副本数量（避免一台宕机），可以不设置，默认值是3 -->
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
 
<!--hdfs 监听namenode的web的地址，默认就是9870端口，如果不改端口也可以不设置 -->
<property>
    <name>dfs.namenode.http-address</name>
    <value>hadoop101:9870</value>
</property>
        
<!-- hdfs保存datanode当前数据的路径，默认值需要配环境变量，建议使用自己创建的路径，方便管理-->
<property>
    <name>dfs.datanode.data.dir</name>
    <value>/usr/hadoop/hadoop-3.2.1/hdfs/data</value>
</property>
 
<!-- hdfs保存namenode当前数据的路径，默认值需要配环境变量，建议使用自己创建的路径，方便管理-->
<property>
    <name>dfs.namenode.name.dir</name>
    <value>/usr/hadoop/hadoop-3.2.1/hdfs/name</value>
</property>
 
```

#### 4. yarn-site.xml

```xml
<!-- 必须配置指定YARN的老大（ResourceManager）在哪一台主机 -->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoop101</value>
</property>
 
<!-- 必须配置提供mapreduce程序获取数据的方式 默认为空 -->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
```

#### 5. mapred-site.xml

```xml
<!-- 必须设置，mapreduce程序使用的资源调度平台，默认值是local，若不改就只能单机运行，不会到集群上了 -->
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<!-- 这是3.2以上版本需要增加配置的，不配置运行mapreduce任务可能会有问题，记得使用自己的路径 -->
<property>
    <name>mapreduce.application.classpath</name>
    <value>
        /usr/hadoop/hadoop-3.2.1/etc/hadoop,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/common/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/common/lib/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/hdfs/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/hdfs/lib/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/mapreduce/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/mapreduce/lib/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/yarn/*,
        /usr/hadoop/hadoop-3.2.1/share/hadoop/yarn/lib/*
    </value>
</property>
```

#### 6. 配置workers

配置需要启动datanode节点的主机

```xml
hadoop101
hadoop102
hadoop103
```

#### 7. 分发配置文件

将配置好的文件分发到主机hadoop102和hadoop103上

```shell
xsync /usr/hadoop/hadoop-3.2.1/etc/hadoop/
```

#### 8. 启动集群

在/hadoop/sbin路径下，将以下参数添加到start-dfs.sh和stop-dfs.sh的顶部

```shell
HDFS_DATANODE_USER=root
HADOOP_SECURE_DN_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root
```

将以下参数添加到start-yarn.sh和stop-yarn.sh的顶部

```shell
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
```

启动hdfs

```shell
root@Hadoop101:~# start-dfs.sh
```