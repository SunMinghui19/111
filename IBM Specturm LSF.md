# IBM Specturm LSF
 IBM Spectrum LSF产品是一套业内领先的系统管理和部署集成软件，在2018年拥有60%世界500强的用户，用户行业遍及各个行业，市场占有率在全球和国内均为第一。  
  
通过LSF驻留程序将硬件资源的运行情况收集起来，在平台层面实现统一监控和管理。从用户的角度来看，他们看到的不再是大批的服务器，而是“一台”机器，管理难度和相应的工作量得以大大降低。
在此基础上，根据不同的调度策略和不同的排队机制，该产品能够做到在同样的时间里，完成更多的作业任务。对要求苛刻的分布式HPC环境来说，它能够提供策略驱动的全面的智能调度功能集，支持用户全面利用计算资源，并确保最优的应用程序性能。IBM Spectrum LSF是实现将几百台机器当做一台进行管理的核心，它还能将出现故障的节点任务转移到其他的节点上去，通过良好的调度增加高性能计算的效率。


# 起源
Platform Load Sharing Facility（简称 LSF）是一个用于作业调度的管理平台，主要应用于高性能计算领域，近年来又广泛应用于金融、工业设计和电子设计自动化（Electronics Design Automation，简称 EDA）等领域。LSF 主要用于处理批处理作业的调度，并且支持不同架构平台的 Unix 和 Windows 操作系统。在 2012 年 1 月，Platform 被 IBM 收购，LSF 也被赋予了新的品牌名称 – IBM Spectrum LSF
# 一、运行原理
  高性能计算的本质，就是在最大程度上提高软硬件资源的利用率。IBM Spectrum LSF 采用的是典型的 Master-Salve 网络架构。Master 节点负责接收用户请求和作业调度。Salve 节点作为计算节点提供计算资源，运行计算作业。LSF Master 节点能够根据集群当前的资源使用情况，将作业调度到满足作业要求的计算节点上去。
  在实际生产环境中IBM LSF 能将多个集群连接在一起，一个集群往往是企业中的一个部门，每个集群中有一个主控机，此主控机负责收集该集群系统中的各台主机的负载信息，并且根据各主机的负载信息情况对作业进行调度。各个集群系统之间根据一定的策略进行资源共享。在每个主控机上定义了该集群能共享哪些集群系统中的资源。当用户发送了一个任务请求时，LSF 系统能将此任务发送到对应的资源所在地，并根据调度策略选择负载较轻的机器对此任务进行处理。当多个用户请求同一个资源时，根据用户的请求优先级来确保优先级高的用户的紧急任务能首先得到满足。


# 二、部署与运行流程

## 处理流程
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/LSF-del.JPG)
* 1、提交一份作业  
  从 LSF 客户端，或者是一个运行 bsub 命令的服务器上提交一份作业，当提交这份作业时，如果不指定哪个队列，这份作业就会被提交到系统默认的队列中，作业在队列中等待安排，这些作业处于等待状态。
* 2、调度作业  
  后台的主进程 mbatchd 将处理队列中的作业，在一个预定的时间间隔里将这些作业按设定的计划，传递给主调度进程 mbschd。主调度进程 mbschd 评估这份工作时，根据作业的优先权制定调度决策、调度机制和可利用资源。主调度进程选择最佳的主机，在哪里作业可以运行，并将它的决策返回给后台主进程 mbatchd。主负载信息管理进程（LIM：Load Information Manager）收集资源信息，主 LIM 与 mbatchd 主进程交流这些信息，反过来 mbatchd 主进程使用之前交流信息支持调度决定。  
* 3、分配作业  
  mbatchd 主进程一收到 mbschd 发过来的决定，立即分配作业到主机。  
* 4、运行作业  
  从属批处理进程（sbatchd），从 mbatchd 主进程接到要求，为这份作业创建一个子 sbatchd 和一个执行环境，通过使用一个远程执行服务器开始这个作业。  
* 5、返回输出  
  当一个作业完成时，如果这个作业没有任何问题，它处于一个完成状态。如果有错误作业无法完成，这份作业处于退出状态。sbatchd 传达作业信息，包括错误提示和给 mbatchd 的输出信息。  
* 6、反馈  
  最后，mbatchd 给提交主机反馈作业输出信息、作业错误、提示信息、作业信息。  
  
## Kubernetes连接器
IBM Spectrum LSF对K8s的连接器使用IBM Spectrum LSF 的调度技术并且集成进了K8s中去。  
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/LSF-k8s.JPG)  
1、LSF调度组件被打包进了容器中，并提供了helm表部署到K8s的环境中  
2、用户通过kubectl提交作业到K8s的API server，如果要想使用LSF调度程序，就必须要修改SchedulerName字段，否则的话Pod将使用默认的Scheduler进行调度。  
3、为了了解Pod和节点的状态，LSF 调度程序使用了一个驱动程序，该驱动程序侦听Kubernetes API服务器并将Pod请求转换为LSF 调度程序中的作业。  
4、一旦LSF 调度程序决定了在何处调度Pod，驱动程序便会将Pod绑定到特定节点。  
5、Kubelet将以正常方式在目标节点上执行和管理Pod生命周期。  

### Kubernetes作业不能使用以下LSF 功能：

* bstop命令
* 支持有限的Bkill命令Bkill可以终止Pod，但不向在Pod内部运行的作业发送任意UNIX信号。
* LSF作业调整大小
* LSF数据管理器
* LSF许可证调度程序
* LSF多集群功能
* LSF 资源连接器
* CPU和NUMA节点关联


具体的配置方案：https://www.ibm.com/support/knowledgecenter/ja/SSWRJV_10.1.0/kubernetes_connector/install_about.html
## 成熟丰富的调度策略
* 基于时间段的自动配置
* 基于服务满足协议的调度
* 高级抢占调度（资源/许可证/CPU）
* 公平竞争调度
* 动态优先级调度
* 限期、强制和独占调度
* 高级资源预留调度
* NUMA和CPU Affinity调度
* Docker容器调度
* 智能网络拓扑结构的调度
* 绿色节能调度

# 三、资源适配
  IBM Spectrum LSF 宣称其管理的集群规模目前已经能够支持多达六千个计算节点。从广义上来讲，LSF 管理的集群就相当于一个小型的企业内部私有云。但是，企业拥有的计算资源毕竟有限，当企业在研发或者业务高峰时期，几千个的计算节点也有可能无法满足大量的用户作业需求。公有云概念的兴起，使得 LSF 的企业用户自然而然的联想，是否可以从各种公有云平台中临时租用虚拟机，作为计算节点加入到 LSF 集群，来满足企业的高峰作业需求。LSF Resource Connector 正是为了满足这一特性需求，设计并实现的一个混合云计算解决方案。
##  Resource Connector 架构
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/RC-structure.JPG)  
  LSF Resource Connector 是 LSF 在 2016 年 7 月发布的最新版本 10.1 里的的一个新特性，该特性使得 LSF 能够根据集群的作业负载情况，从各种外部计算资源管理系统或者公有云平台借用计算资源（虚拟机或者物理机），将其加入 LSF 集群，然后 LSF Master 就可以把作业调度到借用的资源上。

## LSF 如何从 Resource Provider 借用机器
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/RC-unfit.JPG)

```
1.用户像往常一样提交一个 LSF 作业。作业会产生一个对计算资源的需求（demand），但是集群中并没有足够的计算资源来处理作业，这时候就需要从
  外部资源提供方借用资源了。
2.mbatch 守护进程会首先检查是否已经有机器被分配并且能够满足这个需求，如果没有，它会根据外部资源模板（template）的定义，计算针对每个模
  板需要几台机器，然后生成一个具体的资源请求（request），然后把这个 request 发给 ebrokerd 守护进程。
3.管理员需要事先配置一些 template 来定义外部的机器类型。每个外部资源提供方都需要定义自己的 template 配置文件。template 是 LSF 内部的
  资源请求和外部机器的映射桥梁。每个 template 代表一组具有相同属性的机器，比如具有同样的 cpu，相同的内存，相同的操作系统和软件等。
4.根据作业的 demand，LSF Resource Connector 请求外部资源提供方来分配机器。例如，如果资源提供方是 EGO，Resource Connector 就会请求 
  EGO 分配机器。
5.对于 EGO 管理的资源，如果有足够的机器在共享的资源组中可用，那么就会成功的从中借到资源。
6.ebrokerd 守护进程会监控请求的状态，直到它检测到资源提供方成功的分配了机器，并在新分配的机器上作为 LSF Slave 节点启动 LSF 守护进程。
  这时，ebrokerd 就会通知 LSF，这台机器可以加入 LSF 并且可以使用了。
7.机器加入到 LSF 集群以后，作业就会被调度到这台机器上。
8.当没有新的资源需求的时候，LSF 会通知 Resource Connector，同样通过 ebrokerd 守护进程把资源返回给资源提供方。
```

 ![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/%E6%9E%B6%E6%9E%84.JPG)
  
# 六、Qos（服务质量）
##  无语伦比的作业调度性能 - 比开源软件快150倍
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/%E8%B0%83%E5%BA%A61.JPG)
![image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/LSF/%E8%B0%83%E5%BA%A62.JPG)





# 参考：
```
spark和IBM LSF 深度集成与实战 https://developer.ibm.com/zh/articles/ba-cn-spark-ibm-lsf-integration/
IBM Developer https://developer.ibm.com/zh/
IBM Spectrum LSF 的混合云解决方案：https://developer.ibm.com/zh/articles/cl-lo-ibm-spectrum-lsf-hybrid-cloud-solution/
IBM LSF 官网 https://community.ibm.com/community/user/legacy


LSF与Kubernetes的连接：https://www.ibm.com/support/knowledgecenter/ja/SSWRJV_10.1.0/lsf_welcome/lsf_kc_kubernetes_connector.html
LSF安装k8s：https://www.ibm.com/support/knowledgecenter/ja/SSWRJV_10.1.0/kubernetes_connector/install_about.html

```
