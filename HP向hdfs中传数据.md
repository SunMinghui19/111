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
## namenode和datanode
集群硬件配置：  
 在hdfs1.0中1个namenode 和多个datanod  
 
 应用：namenode就当一个目录服务器，应用访问的时候都先去访问那么node，获得需要获得的数据都放在那些datanode上面。  
 ### namenode
 元数据：告诉你一个文件被拆分成多少块，每块存在哪。会建立一个映射表，映射表中的数据全都是保存在内存中的
 ## secondarynamenode
 secondarynamenode是namenode的备份  
 ！！！注意：在hdfs1.0中不是热备份，而是冷备份  （热备份：如果namenode出现了问题，secondarynamenode马上顶上来）  
 
 如果在一个小集群中，secondarynamenode可以与namenode放在一起  
 
 
# HDFS
全称：Hadoop Distributed File System  

HDFS的设计目标就是海量数据批处理的需求  

## 块
是HDFS中最核心的概念  
为了分摊磁盘读写开销也就是在大量数据间分摊磁盘寻址的开销  
其中块的大小为64M 128M  
## 名称节点
相当于整个HDFS集群的管家（数据目录）保存元数据
FsImage Editlog（略，未做笔记）

### 元数据
 元数据抱包含以下内容：
 * 文件是什么
 * 文件被分为多少块
 * 每个块和文件是怎么映射的
 * 每个块被存储在那么服务器上面
## 数据节点
存储具体的数据

# HDFS的体系结构
## hdfs命名空间
目录>文件>块
局限性：

# HDFS存储原理
默认冗余因子为3
