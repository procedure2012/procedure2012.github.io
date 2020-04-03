---
title: spark-install-guide
date: 2020-03-30 18:52:03
categories: 科普 
tags: 技术
---
(未完待续)
以下操作没有强调需要用哪个用户说都都可以，强调用root的一定要用root。

## 修改网络配置

### 修改主机名

在每一台主机上修改/etc/hosts,添加IP绑定
```sh
# 192.168.178.176 rasparripi (注释掉这一行)

192.168.178.176 master
192.168.178.85 worker1 
192.168.178.86 worker2
192.168.178.87 worker3
192.168.178.57 brotherhood
```
在对应的主机上修改/etc/hostname，更改主机名和/etc/hosts里面的一致
```sh
# rasparrypi (注释掉原来的主机名)
master
```
重启所有主机使主机名生效
```sh
pi@master 主机名已经生效
```

### 修改ssh登陆方式

在每一台主机上修改/etc/ssh/sshd_config文件，将以下三项开启yes状态
```sh
PermitRootLogin yes
PermitEmptyPasswords yes
PasswordAuthentication yes
```
这样允许以root登录，并开启免密码登录。以后master需要免密登录workers，然后重启ssh服务
```sh
service ssh restart
```

### 添加ssh免密登录

以下操作都用root身份。因为权限问题需要master的root用户和所有worker的root用户通信，建立起hadoop和spark集群。在每一台主机上生成rsa公私钥匙对。
```sh
ssh-keygen -t rsa
```
添加进本机的受信任公钥
```sh
cd ~/.ssh
cp id_rsa.pub authorized_keys
```
检测是否能免密连接到自己。
```sh
ssh localhost
```
将master上的公钥分发到其他主机，使得master可以免密登录其他主机。
```sh
scp id_rsa.pub root@worker1:/home/root
```
登录相应主机root用户
```sh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
rm ~/id_rsa.pub
```
在master上检测是否成功
```sh
ssh worker1
ssh worker2
ssh worker3
ssh brotherhood
```
此时应该是需要输入`yes`以信任远程主机，不需要再输入密码。

## 准备安装包

java1.8
hadoop3.1.3
scala2.12
spark2.4.5-bin-hadoop2.7

## 安装java

在所有主机上将java安装至/opt/。位置不是必须的，根据个人喜好来。
```sh
cp java.tar.gz /opt/
cd /opt/
tar -zxvf java.tar.gz
mv java jdk
```

### 配置环境变量

在每个主机上修改/etc/profile。
```sh
export JAVA_HOME=/opt/jdk
export JRE_HOME=/opt/jdk/jre
export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$JAVA_HOME/bin:$PATH
```
重新载入环境变量，并检查java
```sh
source /etc/profile
java -version
```

## 安装hadoop

在所有主机上将hadoop安装到`/opt/`，因为我安装在了`/opt/`下，所以上边的ssh和之后的很多操作都是以root进行的。
```sh
cp hadoop.tar.gz /opt/
cd /opt/
tar -zxvf hadoop.tar.gz
cd hadoop-3.1.3
mkdir tmp
mkdir dfs
mkdir dfs/name
mkdir dfs/data
mkdir dfs/node
```

### 配置环境变量

在所有主机上修改/etc/profile，添加如下内容
```sh
export HADOOP_HOME=/opt/hadoop-3.0.0
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export PATH=$HADOOP_HOME/bin:$PATH
export PATH=$HADOOP_HOME/sbin:$PATH
```
重新载入环境变量
```sh
source /etc/profile
```

### 在master上配置hadoop

#### 在hadoop-env.sh中添加

```sh
export JAVA_HOME=/opt/jdk
# 因为使用root，所以需要添加对root用户的支持
export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"
```

#### 在core-site.xml中添加

```sh
<configuration>
    <property>  
        <name>fs.defaultFS</name>  
        <value>hdfs://master:9000</value>  
    </property>  
    <property>  
        <name>io.file.buffer.size</name>  
        <value>131072</value>  
    </property>  
    <property>  
        <name>hadoop.tmp.dir</name>  
        <value>file:/opt/hadoop-2.7.7/tmp</value>  
        <description>Abasefor other temporary directories.</description>  
    </property>  
    <property>  
        <name>hadoop.proxyuser.spark.hosts</name>  
        <value>*</value>  
    </property>  
    <property>  
        <name>hadoop.proxyuser.spark.groups</name>  
        <value>*</value>  
    </property> 
</configuration>
```

#### 在hdfs-site.xml中添加

```sh
<configuration>
    <property>  
        <name>dfs.namenode.secondary.http-address</name>  
        <value>master:9001</value>  
    </property>  
    <property>  
        <name>dfs.namenode.name.dir</name>  
        <value>file:/opt/hadoop-2.7.7/dfs/name</value>  
    </property>  
    <property>  
        <name>dfs.datanode.data.dir</name>  
        <value>file:/opt/hadoop-2.7.7/dfs/data</value>  
    </property>  
    <property>  
        <name>dfs.replication</name>  
        <value>3</value>  
    </property>  
    <property>  
        <name>dfs.webhdfs.enabled</name>  
        <value>true</value>  
    </property>  
</configuration>
```

#### 在mapred-site.xml中添加

```sh
<configuration>
    <property>  
        <name>mapreduce.framework.name</name>  
        <value>yarn</value>  
    </property>  
    <property>  
        <name>mapreduce.jobhistory.address</name>  
        <value>master:10020</value>  
    </property>  
    <property>  
        <name>mapreduce.jobhistory.webapp.address</name>  
        <value>master:19888</value>  
    </property>  
</configuration>
```

#### 在yarn-site.xml中添加

```sh
<configuration>
    <property>  
        <name>yarn.nodemanager.aux-services</name>  
        <value>mapreduce_shuffle</value>  
    </property>  
    <property>  
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>  
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>  
    </property>  
    <property>  
        <name>yarn.resourcemanager.address</name>  
        <value>master:8032</value>  
    </property>  
    <property>  
        <name>yarn.resourcemanager.scheduler.address</name>  
        <value>master:8030</value>  
    </property>  
    <property>  
        <name>yarn.resourcemanager.resource-tracker.address</name>  
        <value>master:8035</value>  
    </property>  
    <property>  
        <name>yarn.resourcemanager.admin.address</name>  
        <value>master:8033</value>  
    </property>  
    <property>  
        <name>yarn.resourcemanager.webapp.address</name>  
        <value>master:8088</value>  
    </property>  
</configuration>
```

#### 在workers中添加

```sh
# localhost (注释掉原有的localhost)
worker1
workre2
worker3
brotherhood
```
workers这个文件里说明所有用作datanode的主机名，master只用作namenode所以不需要添加进来。

#### 分发文件

将上述6个文件分发到所有其他主机。只需要分发覆盖，不需要改动，所有主机的hadoop配置都是完全一样的。或者也可以直再master上安装hadoop并配置，然后将整个hadoop-3.1.3发送到其他主机上。
```sh
scp hadoop-env.sh root@worker1:/opt/hadoop-3.1.3/etc/hadoop
# 或者
scp -r hadoop-3.1.3 root@worker2:/opt/
# 或者
scp -r hadoop root@worker2:/opt/hadoop-3.1.3/etc
```

#### 检测hadoop是否能启动

进入hadoop-3.1.3/bin/，格式化新的文件系统
```sh
./hadoop namenode -format
```
进入hadoop-3.3.1/sbin/启动hadoop
```sh
./start-all.sh
```
进入web UI查看是否所有节点都已启动
```sh
http://192.168.178.176:8088/cluster/nodes
http://192.168.178.176:9870
```
使用jps检测所有进程是否正常运行
```sh
master
```
```sh
worker
```

## 安装scala

在每个主机上安装scala到`/opt`，spark使用scala开发，所以一定要安装正确版本。spark2.4.5对应scala2.12.
```sh
cp scala-2.12.tgz /opt
tar -zxvf scala-2.12.tgz
```
配置环境变量`/etc/profile`
```sh
export SCALA_HOME=/opt/scala-2.12
export PATH=$SCALA_HOME/bin:$PATH
```
重新载入环境变量
```sh
source /etc/profile
```

## 安装spark

在每个主机上将spark安装到`/opt`。
```sh
cp spark-2.4.5-bin-hadoop2.7 /opt/
tar -zxvf spark-2.4.5-bin-hadoop2.7
```
在/etc/profile中添加环境变量
```sh
export SPARK_HOME=/usr/local/spark-2.3.0-bin-hadoop2.7
export PATH=$SPARK_HOME/bin:$PATH
```
重新载入环境变量
```sh
source /etc/profile
```

### 在master上配置sprak

编辑slaves.template
```sh
cd $SPARK_HOME/conf
cp slaves.template slaves
```
替换默认内容`localhost`
```sh
worker1
worker2
worker3
brotherhood
```
编辑spark-env.sh
```sh
export SPARK_DIST_CLASSPATH=$(/opt/hadoop-3.1.3/bin/hadoop classpath)
export HADOOP_CONF_DIR=/opt/hadoop-3.1.3/etc/hadoop
export SPARK_MASTER_IP=192.168.178.176
export SPARK_MASTER_HOST=192.168.178.176
```
编辑spark-config.sh
```sh
export JAVA_HOME=/opt/jdk
```

### 分发文件

将conf文件夹分发到所有主机并覆盖原有文件。或者只在master上安装spark然后将整个spark分发到其他主机。
```sh
scp spark-config.sh root@worker1:/opt/spark2.4.5-bin-hadoop2.7/conf
# 或者
scp -r spark2.4.5-bin-hadoop2.7 root@worker2:/opt/
# 或者
scp -r spark2.4.5-bin-hadoop2.7/conf root@worker2:/opt/spark2.4.5-bin-hadoop2.7
```

### 检测spark是否能启动

运行spark检测安装结构
```sh
 #需要先运行hadoop
cd $SPARK_HOME/sbin
./start-all.sh
```
登录web UI查看是否正常
```sh
http://192.168.178.176:8080
```
用jsp查看进程是否正常
```sh
jsp
master
```
```sh
jsp
woreker
```
运行`spark2.4.5-bin-hadoop2.7/examples/src/main/python`中的一个例子检测是否正常
```sh
spark-submit --master spark://192.168.178.176:7077 pi.py 3
```

---

参考：
[搭建Spark集群](https://www.jianshu.com/p/eca55698b6db)
[Hadoop集群+Spark集群搭建（一篇文章就够了）](https://www.cnblogs.com/zhangyongli2011/p/10572152.html)

