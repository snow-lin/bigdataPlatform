

0.环境配置

# 若未安装过可省略此步
sudo apt-get remove docker docker-engine docker.io
# 安装最新的docker,脚本安装,自动选择最新版
# 如果已有docker,可忽略
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh
curl -sSL https://get.docker.com/ | sh 
# 确认Docker成功安装
sudo docker run hello-world

# 安装docker-compose
# 第一种,直接从github上下载,速度较慢
sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
# 第二种,daocloud下载
sudo curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
# 添加可执行权限
sudo chmod +x /usr/local/bin/docker-compose
# 测试安装结果
docker compose version

1.目录结构

docker-omit/
├── apache-zookeeper-3.8.0-bin.tar.gz
├── delete.sh
├── docker-compose.yml
├── Dockerfile-hadoop 
├── Dockerfile-hbase
├── Dockerfile-jdk
├── Dockerfile-zookeeper
├── etc
│   ├── hadoop
│   │   ├── core-site.xml
│   │   ├── hdfs-site.xml
│   │   ├── mapred-site.xml
│   │   ├── workers
│   │   └── yarn-site.xml
│   ├── hbase
│   │   ├── backup-masters
│   │   ├── hbase-site.xml
│   │   ├── regionservers
│   │   └── start-hh.sh
│   └── zookeeper
│       ├── start-zookeeper.sh
│       ├── zk1
│       │   ├── conf
│       │   │   └── zoo.cfg
│       │   └── data
│       │       └── myid
│       ├── zk2
│       │   ├── conf
│       │   │   └── zoo.cfg
│       │   └── data
│       │       └── myid
│       ├── zk3
│       │   ├── conf
│       │   │   └── zoo.cfg
│       │   └── data
│       │       └── myid
│       └── zoo.cfg
├── hadoop-3.3.3.tar.gz
├── hbase-2.5.3-bin.tar.gz
└── jdk-8u162-linux-x64.tar.gz

2.完整指令流程

  # 创建docker目录
  mkdir docker
  cd docker
  # 创建配置文件目录
  mkdir -p ./etc/hadoop
  mkdir -p ./etc/hbase
  mkdir -p ./etc/zookeeper
  # 下载软件压缩包
  # 如果更换下载源,或者下载版本,需修改对应的XX_UNZIP参数
  # 方式一:直接在对应的Dockerfile文件里面修改
  # 方式二:在docker build阶段通过--arg参数传入
  wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz
  wget https://mirror.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.5/hadoop-3.3.5.tar.gz
  wget https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/2.5.8/hbase-2.5.8-bin.tar.gz
  wget -c http://res.aihyzh.com/大数据技术原理与应用3/02/jdk-8u162-linux-x64.tar.gz #下载资源
  wget https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-3.4.2/spark-3.4.2-bin-hadoop3.tgz
  # 编写hadoop配置,具体配置文件在下面
  vim ./etc/hadoop/core-site.xml
  vim ./etc/hadoop/hdfs-site.xml
  vim ./etc/hadoop/mapred-site.xml
  vim ./etc/hadoop/yarn-site.xml
  vim ./etc/hadoop/workers
  # 编写hbase配置和脚本,具体文件在下面
  vim ./etc/hbase/hbase-site.xml
  vim ./etc/hbase/regionservers
  vim ./etc/hbase/backup-masters
  vim ./etc/hbase/start-hh.sh
  # 编写zookeeper配置和脚本,具体文件在下面
  vim ./etc/zookeeper/zoo.cfg
  vim ./etc/zookeeper/start-zookeeper.sh
  # 编写spark配置和脚本,具体文件在下面
  vim ./etc/spark/spark-env.sh
  vim ./etc/spark/spark-defaults.conf
  vim ./etc/spark/workers
  # 编写四个镜像的Dockerfile文件
  vim Dockerfile-jdk
  vim Dockerfile-hadoop
  vim Dockerfile-hbase
  vim Dockerfile-zookeeper
  vim Dockerfile-spark
  # 编写docker-compose.yml集群文件
  vim docker-compose.yml
  # 拉取基础镜像
  docker pull ubuntu
  # 创建四个镜像
  docker build -f Dockerfile-jdk --tag omit-jdk-1.8 .
  docker build -f Dockerfile-hadoop --tag omit-hadoop-3.3.5 .
  docker build -f Dockerfile-hbase --tag omit-hbase-2.5.8 .
  docker build -f Dockerfile-zookeeper --tag omit-zookeeper-3.8.4 .
  docker build -f Dockerfile-zookeeper --tag omit-spark-3.4.2 .
  # 启动集群
  docker compose up -d
  # 查看成功启动的容器
  docker ps
  # 查看所有容器,如果有启动失败的,将会在这里看到
  docker ps -a
  # 进入master1节点
  docker exec -it docker-master1 bash
  # 初始化hdfs,启动hdfs和hbase
  /start-hh.sh
  # 查看当前节点的进程
  jps
  # 启动spark和spark-history服务
  # 假设当前目录为/app
  ./spark/sbin/start-all.sh
  ./spark/sbin/start-history-server.sh
  # 查看其他节点的进程
  ssh master2(worker1,worker2...worker5)
  jps
  # 返回上一个节点
  exit
  # ctrl shift t 新开一个窗口
  # 进入zk1节点
  docker exec -it docker-zk1 bash
  # 查看当前节点的集群是follower还是leader
  ./zookeeper/bin/zkServer.sh status
  # 返回
  exit 

3.Dockerfile文件



image:omit-jdk-1.8
Dockerfile-jdk:

FROM ubuntu:22.10

# 软件目录,绝对路径
ARG SOFT_PATH=/app

# 压缩包名
ARG JAVA_PACKAGE=jdk-8u162-linux-x64.tar.gz

# 压缩包解压后目录名
ARG JAVA_UNZIP=jdk1.8.0_162

# 目录改名后,即以后的目录名
ARG JAVA_CHANGE=jdk

SHELL ["bash", "-c"]

RUN sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
    apt-get clean && \apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \wget \ssh && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*



# ssh免登录设置
RUN echo "/etc/init.d/ssh start" >> ~/.bashrc && \
    ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa && \
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys && \
    chmod 0600 ~/.ssh/authorized_keys



# 环境变量
ARG JAVA_HOME=${SOFT_PATH}/${JAVA_CHANGE}

# 设置工作目录
WORKDIR $SOFT_PATH

# 复制并解压文件
ADD $JAVA_PACKAGE $SOFT_PATH

# 修改目录名并设置环境变量
RUN mv $JAVA_UNZIP $JAVA_CHANGE &&\
    echo export JAVA_HOME=${JAVA_HOME} >> ~/.bashrc &&\
    echo export PATH=\$PATH:\$JAVA_HOME/bin >> ~/.bashrc &&\
    echo export JAVA_HOME=${JAVA_HOME} >> /etc/profile &&\
    echo export PATH=\$PATH:\$JAVA_HOME/bin >> /etc/profile &&\
    sed -i '1i source /etc/profile' ~/.bashrc &&\
    source ~/.bashrc






image:omit-hadoop-3.3.5
Dockerfile-hadoop:

FROM omit-jdk-1.8

# 软件目录,绝对路径
ARG SOFT_PATH=/app

# hdfs的namenode和datanode数据存放路径
ARG DATA_DIR=$SOFT_PATH/hdfs

# 配置文件在主机当前目录下的相对路径
ARG HOST_CONF=./etc/hadoop

# 压缩包名
ARG HADOOP_PACKAGE=hadoop-3.3.5.tar.gz

# 压缩包解压后目录名
ARG HADOOP_UNZIP=hadoop-3.3.5

# 目录改名后,即以后的目录名
ARG HADOOP_CHANGE=hadoop

# 环境变量
ARG HADOOP_HOME=${SOFT_PATH}/${HADOOP_CHANGE}

# 在最底层镜像中设置的路径
ARG JAVA_HOME=/app/jdk

# 创建目录
WORKDIR $DATA_DIR
# 设置工作目录
WORKDIR $SOFT_PATH

# 复制并解压文件
ADD $HADOOP_PACKAGE $SOFT_PATH

# 修改目录名并设置环境变量
RUN mv $HADOOP_UNZIP $HADOOP_CHANGE &&\
    echo export HADOOP_HOME=${HADOOP_HOME} >> /etc/profile &&\
    echo export PATH=\$PATH:\$HADOOP_HOME/bin >> /etc/profile &&\
    source /etc/profile

# 拷贝hadoop配置文件
COPY $HOST_CONF/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml
COPY $HOST_CONF/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml
COPY $HOST_CONF/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml
COPY $HOST_CONF/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml
COPY $HOST_CONF/workers $HADOOP_HOME/etc/hadoop/workers

# 设置hadoop环境变量
RUN echo export JAVA_HOME=$JAVA_HOME >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export HADOOP_MAPRED_HOME=$HADOOP_HOME >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export HDFS_NAMENODE_USER=root >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export HDFS_DATANODE_USER=root >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export HDFS_SECONDARYNAMENODE_USER=root >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export YARN_RESOURCEMANAGER_USER=root >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh && \
    echo export YARN_NODEMANAGER_USER=root >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh

# NameNode WEB UI服务端口
EXPOSE 9870

# nn文件服务端口
EXPOSE 9000

# dfs.namenode.secondary.http-address
EXPOSE 9868

# dfs.datanode.http.address
EXPOSE 9864

# dfs.datanode.address
EXPOSE 9866







image:omit-hbase-2.5.8
Dockfile-hbase:
FROM omit-hadoop-3.3.5

# 软件目录,绝对路径
ARG SOFT_PATH=/app

# 底层镜像中的配置
ARG JAVA_HOME=/app/jdk
ARG HADOOP_HOME=/app/hadoop

# 配置文件在主机当前目录下的相对路径
ARG HOST_CONF=./etc/hbase

# 压缩包名
ARG HBASE_PACKAGE=hbase-2.5.8-bin.tar.gz

# 压缩包解压后目录名
ARG HBASE_UNZIP=hbase-2.5.8

# 目录改名后,即以后的目录名
ARG HBASE_CHANGE=hbase

# 环境变量
ARG HBASE_HOME=${SOFT_PATH}/${HBASE_CHANGE}

# 设置工作目录
WORKDIR $SOFT_PATH

# 复制并解压文件
ADD $HBASE_PACKAGE $SOFT_PATH

# 修改目录名并设置环境变量
RUN mv $HBASE_UNZIP $HBASE_CHANGE &&\
    echo export HBASE_HOME=${HBASE_HOME} >> /etc/profile &&\
    echo export PATH=\$PATH:\$HBASE_HOME/bin >> /etc/profile &&\
    source /etc/profile

# 拷贝hbase配置文件
COPY $HOST_CONF/hbase-site.xml $HBASE_HOME/conf/hbase-site.xml
COPY $HOST_CONF/regionservers $HBASE_HOME/conf/regionservers
# COPY $HOST_CONF/backup-masters $HBASE_HOME/conf/backup-masters
COPY $HOST_CONF/start-hh.sh /start-hh.sh

# 设置hbase配置文件,给脚本执行权限
RUN echo export JAVA_HOME=$JAVA_HOME >> $HBASE_HOME/conf/hbase-env.sh &&\
    echo export HBASE_CLASSPATH=$HADOOP_HOME/etc/hadoop >> $HBASE_HOME/conf/hbase-env.sh &&\
    echo export HBASE_MANAGES_ZK=false >> $HBASE_HOME/conf/hbase-env.sh &&\
    echo export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP="true" >> $HBASE_HOME/conf/hbase-env.sh &&\
    chmod +x /start-hh.sh

# 暴露 HBase 监听端口
EXPOSE 16000 16010 16020 16030








image:omit-zookeeper-3.8.4
Dockerfile-zookeeper:
FROM omit-jdk-1.8

# entrypoint不会在shell环境中执行，不会继承继承镜像的JAVA_HOME，故而需要显性设置JAVA_HOME
ENV JAVA_HOME=/app/jdk

# zoo.cfg的datadir
ARG DATA_DIR=/data/zookeeper

# hbase-site.xml,hbase在zookeeper上数据的根目录znode节点
ARG DATA_HBASE=/data/hbase

# 配置文件在主机当前目录下的相对路径
ARG HOST_CONF=./etc/zookeeper

# 软件目录,绝对路径
ARG SOFT_PATH=/app

# 配置文件在主机当前目录下的相对路径
ARG HOST_CONF=./etc/zookeeper

# 压缩包名
ARG ZOOKEEPER_PACKAGE=apache-zookeeper-3.8.4-bin.tar.gz

# 压缩包解压后目录名
ARG ZOOKEEPER_UNZIP=apache-zookeeper-3.8.4-bin

# 目录改名后,即以后的目录名
ARG ZOOKEEPER_CHANGE=zookeeper

# 环境变量
ARG ZOOKEEPER_HOME=${SOFT_PATH}/${ZOOKEEPER_CHANGE}

# 创建目录
WORKDIR $DATA_DIR
WORKDIR $DATA_HBASE

# 设置工作目录
WORKDIR $SOFT_PATH

# 复制并解压文件
ADD $ZOOKEEPER_PACKAGE $SOFT_PATH

# 修改目录名并设置环境变量
RUN mv $ZOOKEEPER_UNZIP $ZOOKEEPER_CHANGE &&\
    echo export ZOOKEEPER_HOME=${ZOOKEEPER_HOME} >> /etc/profile &&\
    echo export PATH=\$PATH:\$ZOOKEEPER_HOME/bin >> /etc/profile &&\
    source /etc/profile

# 拷贝zookeeper配置文件
COPY $HOST_CONF/zoo.cfg $ZOOKEEPER_HOME/conf/zoo.cfg
COPY $HOST_CONF/start-zookeeper.sh /start-zookeeper.sh

# 运行zookeeper集群
RUN chmod +x /start-zookeeper.sh
ENTRYPOINT ["/start-zookeeper.sh"]
                                   


image:omit-spark-3.4.2
Dockerfile-spark:
FROM omit-hbase-2.5.8

# 软件目录,绝对路径
ARG SOFT_PATH=/app

# 配置文件在主机当前目录下的相对路径
ARG HOST_CONF=./etc/spark

# 压缩包名
ARG SPARK_PACKAGE=spark-3.4.2-bin-hadoop3.tgz

# 压缩包解压后目录名
ARG SPARK_UNZIP=spark-3.4.2-bin-hadoop3

# 目录改名后,即以后的目录名
ARG SPARK_CHANGE=spark

# 环境变量
ARG SPARK_HOME=${SOFT_PATH}/${SPARK_CHANGE}

# 在最底层镜像中设置的路径
ARG JAVA_HOME=/app/jdk

# 设置工作目录
WORKDIR $SOFT_PATH

# 复制并解压文件
ADD $SPARK_PACKAGE $SOFT_PATH

# 修改目录名并设置环境变量
RUN mv $SPARK_UNZIP $SPARK_CHANGE

# 拷贝spark配置文件
COPY $HOST_CONF/spark-env.sh $SPARK_HOME/conf/spark-env.sh
COPY $HOST_CONF/spark-defaults.conf $SPARK_HOME/conf/spark-defaults.conf 
COPY $HOST_CONF/workers $SPARK_HOME/conf/workers

4.docker-compose集群部署配置文件

docker-compose.yml:

version: '3'
services:
    zk1:
        image: omit-zookeeper-3.8.4
        restart: always
        container_name: docker-zk1
        hostname: zk1
        tty: true
        privileged: true
        environment:
          ZOO_MY_ID: 1
        volumes:
         - ./etc/zookeeper/zk1/data:/data/zookeeper
         - ./etc/zookeeper/zk1/conf:/conf
    zk2:
        image: omit-zookeeper-3.8.4
        restart: always
        container_name: docker-zk2
        hostname: zk2
        tty: true
        privileged: true
        environment:
          ZOO_MY_ID: 2
        volumes:
         - ./etc/zookeeper/zk2/data:/data/zookeeper
         - ./etc/zookeeper/zk2/conf:/conf
    zk3:
        image: omit-zookeeper-3.8.4
        restart: always
        container_name: docker-zk3
        hostname: zk3
        tty: true
        privileged: true
        environment:
          ZOO_MY_ID: 3
        volumes:
         - ./etc/zookeeper/zk3/data:/data/zookeeper
         - ./etc/zookeeper/zk3/conf:/conf
    worker1:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        hostname: worker1
        container_name: docker-worker1
        command: bash
    worker2:
        image: omit-spark-3.4.2
        hostname: worker2
        container_name: docker-worker2
        stdin_open: true
        tty: true
        command: bash
    worker3:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        hostname: worker3
        container_name: docker-worker3
        command: bash
    worker4:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        hostname: worker4
        container_name: docker-worker4
        command: bash
    worker5:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        hostname: worker5
        container_name: docker-worker5
        command: bash
    master1:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        command: bash
        hostname: master1
        container_name: docker-master1
        ports:
         - "9000:9000"
         - "9870:9870"
         - "8088:8088"
         - "16010:16010"
         - "16020:16020"
         - "16030:16030"
         - "8080:8080"
        volumes:
         - ./data/master1/data:/data
        # 去掉注释,将会在容器建立后自动运行脚本,不需要要手动运行
        # 但是,master1和master2节点进程有问题,worker1...5不受影响
        #entrypoint: /start-hh.sh
    master2:
        image: omit-spark-3.4.2
        stdin_open: true
        tty: true
        hostname: master2
        container_name: docker-master2
        command: bash
        volumes:
         - ./data/master2/data:/data





5.配置文件和脚本

core-site.xml:

<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  hdfs-site.xml
  http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<!-- Put site-specific property overrides in this file. -->
<configuration>
  <property>
    <!-- 设置NameNode对外提供HDFS服务 -->
    <name>fs.defaultFS</name>
    <value>hdfs://master1:9000</value>
  </property>
  <property>
    <!-- 指明是否开启用户认证 -->
    <name>hadoop.security.authorization</name>
    <value>false</value>
  </property>
</configuration>



hdfs-site.xml:
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<!-- Put site-specific property overrides in this file. -->
<configuration>
  <property>
    <!-- 文件副本数量, 默认是3 -->
    <name>dfs.replication</name>
    <value>5</value>
    <!-- <value>1</value>-->
  </property>
  <property>
    <!-- 是否启用文件操作权限, 不启用可以以普通账户写操作HDFS文件和目录 -->
    <name>dfs.permissions</name>
    <value>false</value>
  </property>
  <property>
    <!-- NameNode存储数据的文件所在的路径 -->
    <name>dfs.namenode.name.dir</name>
    <value>/app/hdfs/namenode</value>
  </property>
  <property>
    <!-- DataNode存储数据的文件路径 -->
    <name>dfs.datanode.data.dir</name>
    <value>/app/hdfs/datanode</value>
  </property>
  <property>
    <!-- 设置SecondNameNode节点地址 -->
    <name>dfs.namenode.secondary.http-address</name>
    <value>worker1:50090</value>
  </property>
</configuration>


mapred-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<!-- Put site-specific property overrides in this file. -->
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.application.classpath</name>
    <value>
      $HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/ma
      preduce/lib/*
    </value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=/app/hadoop</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=/app/hadoop</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=/app/hadoop</value>
  </property>
</configuration>



yarn-site.xml
<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<configuration>
  <!-- Site specific YARN configuration properties -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>
      JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,
      CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME
    </value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>master1:8032</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>master1:8030</value>
  </property>
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>master1:8031</value>
  </property>
</configuration>



# 设置hadoop集群的datanode节点
workers:
worker1
worker2
worker3
worker4
worker5

hbase-site.xml:
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
          <name>hbase.rootdir</name>
          <value>hdfs://master1:9000/hbase</value>
  </property>


  <!--超时时间-->
  <property>
          <name>zookeeper.session.timeout</name>
          <value>120000</value>
  </property>


  <!--zookeeper集群配置,如果是集群，则添加其他主机地址-->
<property>
          <name>hbase.zookeeper.quorum</name>
          <value>zk1:2181,zk2:2181,zk3:2181</value>
  </property>

  <!--集群或者单机模式,false是单机模式，true是分布式-->
  <property>
          <name>hbase.cluster.distributed</name>
          <value>true</value>
  </property>

  <!--临时文件的目录路径-->
  <property>
    <name>hbase.tmp.dir</name>
    <value>tmp</value>
  </property>

  <!--hbase在zookeeper上数据的根目录znode节点-->
  <property>
          <name>hbase.znode.parent</name>
      <value>/data/hbase</value>
  </property>

  <!--使用本地文件系统设置为false，使用hdfs设置为true-->
  <property>
       <name>hbase.unsafe.stream.capability.enforce</name>
       <value>true</value>
  </property>

</configuration> 

# 设置hbase集群的regionserver节点
regionservers:
worker1
worker2
worker3
worker4
worker5

# 设置hbase集群hmaster的备用节点
backup-masters:
master2

# 用于初始hdfs并启动hdfs和hbase
start-hh.sh:
#!/bin/bash

# 在docker-compose.yml中调用，因为只需要启动一次

# 环境变量没有执行
source profile

# 初始化hdfs
$HADOOP_HOME/bin/hdfs namenode -format

# 启动hdfs
$HADOOP_HOME/sbin/start-all.sh

# 启动hbase
$HBASE_HOME/bin/start-hbase.sh 


zoo.cfg:
# 存储快照文件snapshot的目录（相当于redis的rdb)
dataDir=/data/zookeeper

# ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置的。例如，session的最小超时时间是2*tickTime
tickTime=10000

# 初始值显示的刻度数，同步阶段可能需要
initLimit=10

# 发送请求和获得确认之间可以传递的心跳数
syncLimit=5

# 暴露服务端对外端口
clientPort=2181

# 单个客户端与单台服务器之间的连接数的限制，是ip级别的，默认是60，如果设置为0，那么表明不作任何限制
maxClientCnxns=60

# 需要保留的文件数目,默认是保留3个
autopurge.snapRetainCount=3

# ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能
autopurge.purgeInterval=1

quorumListenOnAllIPs=true

# 监控指标
metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
metricsProvider.httpPort=7000
metricsProvider.exportJvmInfo=true

# 集群配置
# server.{myid}={hostname或者ip}:2888:3888
# 2888->客户端端口
# 3888->选举端口
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888





# 用于启动zookeeper集群                  
start-zookeeper.sh:
#!/bin/bash

echo "Starting ZooKeeper cluster..."

# 将ZOO_MY_ID的值保存到zoo.cfg中设置的dataDir目录的myid文件中
echo "${ZOO_MY_ID}" > /data/zookeeper/myid

# zookeeper的路径，在Dockfile-zookeeper中调用
# zkServer.sh start 启动当前节点
# zkServer.sh start-foreground 启动当前节点并显示调试信息
# zkServer.sh restart 重启当前节点
# zkServer.sh stop 停止当前节点
/app/zookeeper/bin/zkServer.sh start-foreground



spark-defaults.conf:

spark.eventLog.enabled  true
# 需要关注hdfs对外提供服务的端口是不是9000
spark.eventLog.dir      hdfs://master1:9000/spark_log
spark.eventLog.compress true


workers：
# 配置从节点
worker1
worker2
worker3
worker4
worker5


 spark-env.sh：

# 指定 Java Home
export JAVA_HOME=/app/jdk

# 指定 Spark Master 地址
# SPARK_MASTER_HOST和SPARK_DAEMON_JAVA_OPTS中的zookeeper选举只能有一个生效
# export SPARK_MASTER_HOST=node01
export SPARK_MASTER_PORT=7077

# 指定 Spark History 运行参数，需注意hdfs的对外端口
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=4000 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://master1:9000/spark_log"

# 指定 Spark 运行时参数
 export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER-Dspark.deploy.zookeeper.url=zk1:2181,zk2:2181,zk3:2181 -Dspark.deploy.zookeeper.dir=/spark"

