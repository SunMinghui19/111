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
为了分摊磁盘读写开销也就是在大量数据间分摊磁盘寻址的开销,即最小化寻址开销  
其中块的大小为64M，在hadoop2.0后为128M
寻找数据的话要经过三级寻址  
```
找到元数据目录->找到数据节点->从数据节点中取数据
```
这个块也不能无限的大，MapReduce中的map任务通常一次只处理一个块中的数据。如果块太大，那么可能会导致任务数太少（少于集群中的节点数量），作业的运行速度就会比较慢。

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
因为hdfs写入datanode的策略。在k8s上布了10个datanode，去测试它的调度策略是否成立。
```
### 数据读取
* HDFS提供了一个API可以确定一个数据节点所属的机架ID，客户端可以调用API获取自己所属的机架ID
* 当客户端读取数据时，从名称节点获得数据块不同副本的存放位置列表，列表中包含了副本所在的数据节点，可以调用API来确定客户端和这些数据节点所属的机架ID，当发现某个数据块副本对应的机架ID和客户端对应的机架ID相同时就优先选择该副本读取数据，如果没有发现，就随机选择一个副本读取数据

### 数据的错误和恢复
通过校验码进行验证，此处未过多了解

# HDFS的读写过程
## HDFS 读过程java代码
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


## HDFS写数据java代码
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
### shell命令
（1）查看帮助
```
        hdfs dfs -help 
```
 
（2）查看当前目录信息
```
    hdfs dfs -ls /
```
（3）上传文件
```
    hdfs dfs -put /本地路径 /hdfs路径
```
（4）剪切文件
```
    hdfs dfs -moveFromLocal a.txt /aa.txt
```
（5）下载文件到本地
```
    hdfs dfs -get /hdfs路径 /本地路径
```
（6）合并下载
```
    hdfs dfs -getmerge /hdfs路径文件夹 /合并后的文件
```
（7）创建文件夹
```
    hdfs dfs -mkdir /hello
```
（8）创建多级文件夹
```
    hdfs dfs -mkdir -p /hello/world
```
（9）移动hdfs文件
```
    hdfs dfs -mv /hdfs路径 /hdfs路径
```
（10）复制hdfs文件
```
    hdfs dfs -cp /hdfs路径 /hdfs路径
```
（11）删除hdfs文件
```
    hdfs dfs -rm /aa.txt
```
（12）删除hdfs文件夹
```
    hdfs dfs -rm -r /hello
```
（13）查看hdfs中的文件
```
    hdfs dfs -cat /文件
    hdfs dfs -tail -f /文件
```
（14）查看文件夹中有多少个文件
```
    hdfs dfs -count /文件夹
```
（15）查看hdfs的总空间
```
    hdfs dfs -df /
    hdfs dfs -df -h /
```
（16）修改副本数 
```
    hdfs dfs -setrep 1 /a.txt
```
常用的两种
### java API





# 在云主机上测试
```
1、使用dd命令创建了大小为64M,128M,256M的随机字符文件，并保存到hadoop的节点中去
2、编写JAVA程序，上传指定大小的文件。将上传时间保存到日志文件中
3、编写shell脚本，先执行java的上传命令，然后等待10s，再删除上传了的文件。将这个过程重复20次
4、测试了两种极端条件下写入datanode并创建指定副本的时间。即3个datanode都在一个节点上跟3个datanode分布在不同节点上
5、将测试好的数据保存到windows上，使用python将其制作为csv数据集
6、使用python中的plt组件，将对比图显示出来。
```



## 1
1、在master节点上执行如下命令
kubectl get pod -n hadoop-1 -o wide 
通过这条命令我们可以获得namenode容器所在的node节点

2、连接到那么namenode所在的节点
使用docker ps 查看namenode容器的ID
使用 docker exec -it [容器镜像ID] /bin/bash 进入namenode





# K8s搭建hadoop平台
具体可参考我的k8s搭建hadoop平台
现在先参考：https://www.cnblogs.com/00986014w/p/9732796.html?tdsourcetag=s_pctim_aiomsg
# 使用python的matplotlib.pyplot作图

## java中计算程序运行时间的方法
```

long startTime = System.currentTimeMillis();    //获取开始时间

doSomething();    //测试的代码段

long endTime = System.currentTimeMillis();    //获取结束时间

System.out.println("程序运行时间：" + (endTime - startTime) + "ms");    //输出程序运行时间
```

## linux中生成指定大小随机内容文件
```
dd if=/dev/urandom of=data.bin bs=100K count=1
```

# hdfs dfsadmin -report 查看集群节点


# 参考


生成指定大小的文件，并且数据随机：https://blog.csdn.net/hknaruto/article/details/106809808
## hadoop
简单之美（深度剖析hdfs的读写过程-有空可以好好读）：http://shiyanjun.cn/archives/942.html
林子雨hdfs实验：http://dblab.xmu.edu.cn/blog/290-2/
知乎上较好的文章：https://zhuanlan.zhihu.com/p/75896909
                https://www.cnblogs.com/kocdaniel/p/11589382.html
## hdfs shell
shell命令比较全：https://www.cnblogs.com/areyouready/p/9783687.html
hadoop的机架感知：https://www.cnblogs.com/bigdata-stone/p/9311851.html
系统的shell指令：https://www.cnblogs.com/hjwq/p/7873641.html
## hadoop java API（k8s）
查看自己的hdfs监听的端口（与常用的9000不同，具体靠看配置）：https://www.cnblogs.com/vanwoos/p/7839123.html
简单的测试hdfs功能：https://www.cnblogs.com/ccskun/p/7820977.html
官网函数参考：http://hadoop.apache.org/docs/r2.7.1/api/index-all.html
我的java程序的模板来源：https://www.cnblogs.com/growth-hong/p/6396332.html
java向日志文件中写日志：https://blog.csdn.net/douxingpeng1/article/details/83050716
可以参考的javaAPI程序：https://blog.csdn.net/vtopqx/article/details/8608878    
                     https://blog.csdn.net/daxiang12092205/article/details/52717470/
查看hdfs存放的副本位置：https://blog.csdn.net/weixin_41255682/article/details/80312478
## python 处理数据
python中提取日志文件保存到excel：https://blog.csdn.net/zhelijun/article/details/102294138?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~first_rank_v2~rank_v25-1-102294138.nonecase
excel中将文本转为数字：https://www.office68.com/excel/batch-convert-text-into-digital.html
## python plot作图的参考
总体讲解：https://www.cnblogs.com/zhanghongfeng/p/8546693.html
添加线条标注：https://www.cnblogs.com/qccc/p/12795205.html
讲的很明白的线条标注：https://zhuanlan.zhihu.com/p/33722621
线条风格：https://blog.csdn.net/Treasure99/article/details/106044114/

