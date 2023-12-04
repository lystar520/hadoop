# 大数据平台Hadoop

## 1.Hadoop概述

### 1.1 大数据的概念

- 数据规模大
- 数据流转快
- 数据多样性强
- 价值密度低

### 1.2  Hadoop起源

#### 1.2.1 Hadoop的理论基础来自于谷歌的三篇论文

- 分布式文件系统(GFS)
- 分布式计算框架MapReduce
- 分布式的结构化数据存储系统Bigtable

#### 1.2.2 Hadoop的版本有 1.x, 2.x, 3.x

1.x =  HDFS + MapReduce

2.x =  HDFS + MapReduce + Yarn

## 2.Hadoop集群搭建

1. ### 虚拟机的准备:

   三台节点主机名: master、 slave1、slave2

   操作系统： CentOS7

   虚拟化软件: VMware 15pro

   ip: 建议设为三个连续地址

   GateWay : 192.168.121.2

   NetMask: 255.255.255.0

   DNS: 8.8.8.8

   修改UUID： 使用脚本命令一键替换

   ```
   sed -i '/UUID=/c\UUID='`uuidgen`'' /etc/sysconfig/network-scripts/ifcfg-ens33
   ```

   

2. ### 免密认证步骤

   #### 分别在三台机器生成rsa的公钥和私钥

   ```
   ssh-keygen -t rsa
   ```

   #### 拷贝公钥到同一台机器

   ```
   ssh-copy-id master
   ```

   #### 复制第一台机器的认证到其他机器

   ```
   scp  /root/.ssh/authorized_keys slave1:/root/.ssh
   scp  /root/.ssh/authorized_keys slave2:/root/.ssh
   ```

   #### 连接测试

   ```
   ssh slave1
   ```

3. ### 时间同步

为了消除时间不同的问题，需要所有虚拟机连接外网完成同步。

使用ntpdate命令同步阿里云时钟服务器，如命令不存在，需要使用yum install ntpdate进行安装。

```
ntpdate ntp4.aliyun.com
```

如果需要定期自动同步，可以在linux中创建一个定时任务。

```
# 分别在三台机器中执行
crontab  -e
*/1 * * * *   /usr/sbin/ntpdate ntp4.aliyun.com
```

### 4. Hadoop的集群部署模式

- 完全分布式模式
- 独立模式
- 伪分布模式

### 5. JDK安装

在master节点下，安装jdk

下载jdk压缩包并解压到虚拟机

```shell
tar -zxvf jdk-8u161-linux-x64.tar.gz -C  /opt/module/
```

改变jdk文件夹名称

```sh
cd /opt/module
mv jdk1.8.0_161  jdk1.8.0
```

配置环境变量

```sh
vi  /etc/profile
# 进入profile文件后，在尾部增加以下配置
export JAVA_HOME=/opt/module/jdk1.8.0
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

验证java环境

```sh
java -version
```

发送安装好的jdk和环境变量文件---> slave1和slave2中。

```sh
# 复制jdk文件
scp -r /opt/module/jdk1.8.0 slave1:/opt/module/jdk1.8.0
scp -r /opt/module/jdk1.8.0 slave2:/opt/module/jdk1.8.0
# 复制/etc/profile
scp /etc/profile slave1:/etc/
scp /etc/profile slave2:/etc/
# 让环境变量生效，分别在slave1和slave2里执行
source /etc/profile
# 分别在slave1和slave2里验证java环境
java -version
```



### 6 Hadoop安装和配置

本教程采用Hadoop2.7.7





#### 6.1 解压安装Hadoop

将hadoop的压缩包上传至master节点的/opt/software路径下

然后解压

```sh
tar -zxvf /opt/software/hadoop-2.7.7.tar.gz -C /opt/module
```

6.2 配置hadoop环境变量

原因： 为了让Linux操作系统知道hadoop的存在，并且可以直接调用hadoop/bin和sbin里面的脚本，所以需要配置环境变量。

首先配置ip映射文件：hosts

```sh
# 在master配置，之后再通过scp发送到slave1和slave2.
vi /etc/hosts
192.168.189.7 master
192.168.189.8 slave1
192.168.189.9 slave2
# 发送
scp /etc/hosts  slave1:/etc/
scp /etc/hosts  slave2:/etc/
```

配置环境变量，/etc/profile文件尾部，增加hadoop的环境变量

```sh
vi /etc/profile
# Hadoop的环境变量
export HADOOP_HOME=/opt/module/hadoop-2.7.7
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin：
```

让环境变量生效

```
source /etc/profile
```

验证hadoop的安装

```
hadoop version
```

检查环境变量

```
echo $PATH
```









6.3 查看Hadoop的安装目录

bin目录：存放hadoop相关服务的脚本，一般使用sbin目录下的脚本；

etc目录：存放hadoop的配置文件；

include：存放和C++相关的头文件；

lib：存放一些库文件；

libexec：各个服务对用的shell配置文件所在的目录；

sbin：存放hadoop的管理脚本；

share：存放相关jar包；



6.4 Hadoop配置文件

- hadoop-env.sh : 在hadoop中记录java的安装目录，即hadoop的内部环境变量

- slaves： 存储节点/工作节点/从节点
- core-site.xml ： Hadoop的核心配置文件
- hdfs-site.xml :   分布式文件系统HDFS的配置文件
- mapred-site.xml: 分布式计算模块mapreduce的配置文件
- yarn-site.xml : 资源管理调度模块yarn的配置文件



| Hostname |               | HDFS主节点 | HDFS存储节点 | Yarn主节点      | Yarn从节点  | HDFS备用主节点    |
| -------- | ------------- | :--------: | ------------ | --------------- | ----------- | ----------------- |
| 主机名   | ip            |  NameNode  | DataNode     | ResourceManager | NodeManager | SecondaryNameNode |
| master   | 192.168.189.7 |     Y      | Y            |                 | Y           |                   |
| slave1   | 192.168.189.8 |            | Y            | Y               | Y           |                   |
| slave2   | 192.168.189.9 |            | Y            |                 | Y           | Y                 |





### 7. 端口映射

通过宿主机去访问虚拟机的端口，需要关闭虚拟机的防火墙

| 名称 | 命令                        |
| ---- | --------------------------- |
| 状态 | systemctl status firewalld  |
| 开启 | systemctl start firewalld   |
| 关闭 | systemctl stop firewalld    |
| 停用 | systemctl disable firewalld |

- 可以通过master节点的ip来访问50070
- 可以通过master的hostname来访问50070，前提需要在宿主机上增加hosts(主机名和ip的映射)
- 在虚拟机网络编辑器中增加端口转发，就可以通过宿主的ip和转发后的端口来访问Hadoop WebUI



在防火墙中增加一条入站规则：





22 : ssh 

50070 : Hadoop Web UI    master节点 -> 本机上的50070 

点击VMware的编辑-> 虚拟网络编辑器



### 8. Hadoop集群测试

#### 8.1 格式化文件系统

初次启动HDFS集群时，需要对主节点进行格式化处理，所有HDFS上的文件会被清空.

```
hdfs namenode -format
```

***注意： 格式化操作一般在第一次启动集群前进行。***

#### 8.2 启动和关闭Hadoop集群

启动集群包含**HDFS集群**和**YARN集群**两个集群框架的启动！作为Hadoop的基础服务。

1. 启动HDFS集群时需要在HDFS的主机点上执行"start-dfs.sh"，或者关闭HDFS集群执行"stop-dfs.sh"

2. 启动YARN集群时需要在YARN的主机点上执行"start-yarn.sh"，或者关闭YARN集群执行"stop-yarn.sh"

3. 同时启动HDFS和YARN的脚本为"start-all.sh"，在新版中被弃用(deprecated)，不推荐使用

4. 如需同时启动，可以自定义启动脚本:(myhadoop.sh)

   ```bash
   #!/bin/bash
   
   if [ $# -lt 1 ]      # 输入的参数数量小于1
   then 
   	echo "NO Args Input..."
   	exit ;
   fi
   
   case $1 in
   "start")
   	echo "================启动 hadoop集群=============="
   
   	echo "----------------启动 hdfs----master为hdfs的主节点---------"
   	ssh master "/opt/module/hadoop/sbin/start-dfs.sh"    
   	echo "----------------启动 yarn ----slave1为yarn的主节点--------"
   	ssh slave1 "/opt/module/hadoop/sbin/start-yarn.sh"
   ;;
   "stop")
   	echo "================ 关闭 hadoop集群=============="
   	 
   	echo "-------------关闭 yarn ---------------"
   	ssh slave1 "/opt/module/hadoop/sbin/stop-yarn.sh"
   	echo "-------------关闭 hdfs----------------"
   	ssh master "/opt/module/hadoop/sbin/stop-dfs.sh"
   ;;
   *)
   	echo "Input Args Error..."     
   ;;
   esac
   
   ```

   **注意：**

   启动服务的顺序应为: 1. HDFS服务  2. YARN服务 

   关闭服务的顺序应为: 1. YARN服务   2. HDFS服务

#### 8.3 通过UI界面查看Hadoop的运行状态

Hadoop2.x 集群正常启动后，默认开放了50070和8088两个端口，分别用于监控HDFS集群和Yarn集群。

访问集群的WebUI之前，必须关闭每个节点的防火墙：



### 9.Hadoop集群应用

9.1 Hadoop经典案例-词频统计：统计每个单词出现的个数

- 以master为例，在master节点上创建一个文件夹用于存放数据源文件：

```shell
mkdir -p /opt/data
```

- 进入data目录，创建一个源文件

```shell
cd /opt/data
vi word.txt
```

- 添加如下数据

```
hello world
hello hadoop
hello gfxy
```

- 在HDFS上创建一个目录

注意: -p 是递归创建

```sh
hadoop fs -mkdir -p /wordcount/input
```

- 创建完成后，可以在网页50070 ，点击browse file system中查看

- 将本地word.txt上传至hfds中的input目录

  ```
  hadoop fs -put /opt/data/word.txt /wordcount/input
  ```

- 运行官方示例程序jar包，实现单词统计，这个包在hadoop的安装目录下的share/hadoop/mapreduce/里

  ```shell
  cd $HADOOP_HOME/share/hadoop/mapreduce/
  ls
  ```

  ![image-20231102092702572](C:\Users\lyon\AppData\Roaming\Typora\typora-user-images\image-20231102092702572.png)

注意:  share/hadoop/里面存放的是官方提供的程序，将来也可以存放自定义程序或者其他框架软件的jar包。

- 使用hadoop jar命令运行程序

  ```
  hadoop jar hadoop-mapreduce-example-2.7.4.jar wordcount /wordcount/input /wordcount/output
  ```

  - hadoop jar 表示要执行一个jar包
  - hadoop-mapreduce-example-2.7.4.jar表示jar包的名称
  - wordcount表示jar包中的单词计数方法名
  - /wordcount/input 表示单词源文件所在路径，也是上述方法的第一个参数
  - /wordcount/output 表示结果文件的输出路径，也是上述方法的第二个参数

- 运行时可以在网页8088中查看运行进度

![image-20231102095035336](C:\Users\lyon\AppData\Roaming\Typora\typora-user-images\image-20231102095035336.png)

- 运行成功后，在网页50070中查看运行结果
  - _SUCCESS为标记文件，标识运行成功
  - part文件为结果，可以使用文本打开

![image-20231102095006069](C:\Users\lyon\AppData\Roaming\Typora\typora-user-images\image-20231102095006069.png)







## 3.Java访问Hadoop

### 3.1 上传文件到HDFS

```
vi /opt/data/test.txt

Ever since you were small....
```

```
# 上传
hadoop fs -put /opt/data/test.txt /
```

### 3.2 创建IDEA的项目： HadoopClient

- File -> new -> Project... -> Maven ->勾选Create from archetype -> maven-archetype-quickstart -> next  

- 在pom.xml 中添加hadoop相关的依赖信息

  - ```xml
       <dependency>
          <groupId>org.apache.hadoop</groupId>
          <artifactId>hadoop-client</artifactId>
          <version>${hadoop.version}</version>
        </dependency>
      
        <dependency>
          <groupId>org.apache.hadoop</groupId>
          <artifactId>hadoop-common</artifactId>
          <version>${hadoop.version}</version>
        </dependency>
      
        <dependency>
          <groupId>org.apache.hadoop</groupId>
          <artifactId>hadoop-hdfs</artifactId>
          <version>${hadoop.version}</version>
        </dependency>
      
        <dependency>
          <groupId>org.apache.hadoop</groupId>
          <artifactId>hadoop-mapreduce-client-common</artifactId>
          <version>${hadoop.version}</version>
        </dependency>
      
    ```

    在\<properties>\</properties>中添加\<hadoop.version>2.7.7</hadoop.version>

- 在src-main-java中创建package: com.gfxy.hdfs

- 在com.gfxy.hdfs包中创建类: JavaNetUrlAccess

  - ```java
    package com.gfxy.hfds;
    
    import org.apache.hadoop.fs.FsUrlStreamHandlerFactory;
    
    import java.io.BufferedReader;
    import java.io.InputStream;
    import java.io.InputStreamReader;
    import java.net.URL;
    
    public class JavaNetUrlAccess {
    
        static {
            URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
        }
    
        public static void main(String[] args) {
            InputStream inputStream = null;
            try {
    
                inputStream = new URL("hdfs://master:9000/test.txt").openStream();
                InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
                BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
    
                String line = null;
                System.err.println("\n\nstart printing test.txt!\n");
                while ( (line = bufferedReader.readLine()) != null){
                    System.err.println(line);
                }
    
                System.err.println("\ndone!\n\n");
                if (bufferedReader != null){
                    bufferedReader.close();
                }
                if (inputStreamReader != null){
                    inputStreamReader.close();
                }
                if (inputStream != null){
                    inputStream.close();
                }
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
    ```

- 执行此程序，就可以通过java获取到HDFS上的文件test.txt的内容

  ## 4. Java访问HDFS目录和文件

  ## 5. MapReduce的实现

  ### 5.1 Mapper的实现

  - 在工作路径中创建warning_data.txt文件

  ```
  vi /opt/data/warning_data.txt
  114588	LA9HIGECXH1HGC001	2018-04-12 15:45:56	2018-04-26 16:34:36	12743	other	0	6	3	0	1
  114588	LA9HIGECXH1HGC001	2018-04-12 15:45:56	2018-04-26 16:34:36	12743	other	0	6	3	0	2
  
  
  ```

  - 新建ReportCount实体类

    



### 注意事项: 

1. 关闭三个节点防火墙，建议永久关闭防火墙 systemctl disable firewalld
2. 192.168.x.x connect to master:9000 fail， 可能需要检查免密ssh登录、/etc/hosts、环境变量
3. 如何通过windows连接hadoop使用了master的主机名，必须在C:\Windows\System32\drivers\etc\hosts中配置ip和主机名的映射，常见于找不到master的错误，检查方法，通过cmd去ping master

4. 环境变量： 用户环境变量：  ~/.bashrc，  系统的环境变量 /etc/profile， 建议只配置其中一个!







# 注意

快捷键：  

在终端中(cmd，命令行, Terminal)，ctrl+a  是调转到行首，ctrl+e 是跳转到行尾



- 解决各个进程没有启动成功的方法：

  - 关掉已经启动的所有进程，stop-all.sh或者stop-dfs.sh和stop-yarn.sh或者kill -9 进程id

  - 检查所有配置文件，HDFS相关配置文件有:core-site.xml，hdfs-site.xml，slaves. Yarn相关配置文件是yarn-site.xml. MapReduce相关的有mapred.site.xml， 与执行自动脚本stop-all.sh有关的是hadoop-env.sh

  - 格式化HDFS，需要在master上执行hdfs namenode -format

  - 重新启动所有进程

    

- 上传文件到hdfs文件系统: 

  - ```
    hadoop fs -put <local> <hdfs>
    ```

  - 提示 file exists错误，上传失败，说明待上传的文件和hdfs上的文件重名!

  - ```
    #解决办法： hadoop fs -put -f <local> <hdfs>
    ```

    





# 英语角

deprecated : 弃用

src :   源地址   , source 

dst :  目标地址 ,  destination

path: 路径

exists:  存在

Directory : 缩写为dir，目录，文件夹

artifact : 工件，人工制品，软件的工作成果

sample: 样例，示例，抽样

