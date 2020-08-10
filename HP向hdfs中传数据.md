具体工作流程
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/hdfs-princple.png)
```
* 1、hdfs客户端向namenode发送上传请求
* 2、namenode收到请求后，检查目录信息
* 3、namenode检查过后，告诉客户端可以上传
* 4、客户端收到允许上传消息，然后发送请求上传block
* 5、namenode收到请求，检查元数据池，返回给客户端可用的datanode信息
* 6、客户端获得datanode信息，然后选择一个datanode建立pineline连接，发送block到第一个datanode
* 7、datanode接收到数据后，发送数据到下一个datanode，然后将数据保存到硬盘，等待完成信息返回
* 8、下一个datanode重复第7步，知道数据保存达到要求副本数量的服务器数量，依次返回确认结果
* 9、向客户端返回数据保存完成结果
```

# HDFS

全称：Hadoop Distributed File System  
HDFS的设计目标就是海量数据批处理的需求  

## HDFS中的一些相关概念
### 块
是HDFS中最核心的概念之一  
为了分摊磁盘读写开销也就是在大量数据间分摊磁盘寻址的开销  
其中块的大小为64M，在hadoop2.0后为128M
寻找数据的话要经过三级寻址  
```
找到元数据目录->找到数据节点->从数据节点中取数据
```
（这个地方最好解释解释好在哪里）
### namenode和datanode
集群硬件配置：  
 在hdfs1.0中1个namenode 和多个datanod  
 
 应用：namenode就当一个目录服务器，应用访问的时候都先去访问那么node，获得需要获得的数据都放在那些datanode上面。  
 #### namenode
 相当于整个HDFS集群的管家（数据目录）保存元数据
 FsImage Editlog（略，未做笔记）
 
 ##### 元数据
 元数据抱包含以下内容：
 * 文件是什么
 * 文件被分为多少块
 * 每个块和文件是怎么映射的
 * 每个块被存储在那么服务器上面
 告诉你一个文件被拆分成多少块，每块存在哪......。会建立一个映射表，映射表中的数据全都是保存在内存中的
 
 #### secondarynamenode
 secondarynamenode是namenode的备份  
 ！！！注意：在hdfs1.0中不是热备份，而是冷备份  （热备份：如果namenode出现了问题，secondarynamenode马上顶上来）  
 
 如果在一个小集群中，secondarynamenode可以与namenode放在一起  
#### datanode
存储具体的数据


## HDFS存储
默认冗余因子为3，即备份3份
### 数据写入
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/raplicas-rack.JPG)
第一块来了之后要放3份，第一副本放在上传文件的数据节点。（如果是集群外部发出的请求，会随机挑一个磁盘不太慢，内存不满的节点）
第二副本放在不同机架上
第三个副本放在本机架的其他节点
```
因为写入datanode的策略，是否有调度的必要性。测试是否是一个node就是一个机架
```
### 数据读取
* HDFS提供了一个API可以确定一个数据节点所属的机架ID，客户端可以调用API获取自己所属的机架ID
* 当客户端读取数据时，从名称节点获得数据块不同副本的存放位置列表，列表中包含了副本所在的数据节点，可以调用API来确定客户端和这些数据节点所属的机架ID，当发现某个数据块副本对应的机架ID和客户端对应的机架ID相同时就优先选择该副本读取数据，如果没有发现，就随机选择一个副本读取数据

### 数据的错误和恢复
通过校验码进行验证，此处未过多了解

# HDFS的读写过程
## HDFS 读过程代码
读取数据的代码
```
import java.io.BufferedReader;
import java.io.InputStreamReader;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.fs.FSDataInputStream;
public class Chapter3{
    try{
        Configuration conf = new Configuration();
        FileSystem fs = FileSystem.get(conf);
        Path filename = new Path("hdfs://localhost:9000/user/hadoop/test.txt")
        FSDataInputStream is = fs.open(filename);
        BufferedReader d = new BufferedReader(new InputStreamReader(is));
        String content = dreadLine();//读取文件一行
        System.out.println(content);
        d.close();//关闭文件
        fs.close();//关闭hdfs
        } cath (Exception e){
            e.printStackTrace();
        }
}
```
## HDFS读的过程
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/hdfs-open.JPG)


## HDFS写数据过程
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/hdfs-write.JPG)


## HDFS写数据代码
```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.Path;
public class Chapter3{
    try{
        Configuration conf = new Configuration();
        FileSystem fs = FileSystem.get(conf);
        byte[] buff = "Hello world".getBytes();//要写入的内容
        String filename = "hdfs://localhost:9000/user/hadoop/test.txt" //要写入的文件名
        FSDataOuptStream os = fs.create(new Path(filename));
        os.write(buff,0,buff,length);
        System.out.println("Create:"+filename);
        d.close();//关闭文件
        fs.close();//关闭hdfs
        } cath (Exception e){
            e.printStackTrace();
        }
}

```
## HDFS的基本编程方法

常用的两种
* java API
* shell命令


# 在云主机上测试

## 1
1、在master节点上执行如下命令
kubectl get pod -n hadoop-1 -o wide 
通过这条命令我们可以获得namenode容器所在的node节点

2、连接到那么namenode所在的节点
使用docker ps 查看namenode容器的ID
使用 docker exec -it [容器镜像ID] /bin/bash 进入namenode


## 添加datanode到hdfs



不管怎样：https://www.cnblogs.com/ccskun/p/7820977.html在这个网址下的操作我都可以正常的执行

9000端口没有被监听：https://www.cnblogs.com/vanwoos/p/7839123.html

hdfs dfs -ls /user/root/  与 hdfs dfs -ls hdfs://localhost:8020/ 效果相同

生成指定大小的文件：https://blog.csdn.net/keye_2014/article/details/40920065
生成指定大小的文件，并且数据随机：https://blog.csdn.net/hknaruto/article/details/106809808

对于复制文件到hdfs java api 具有很高的参考价值https://www.cnblogs.com/growth-hong/p/6396332.html
