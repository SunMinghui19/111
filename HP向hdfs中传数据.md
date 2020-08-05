具体工作流程
```
1[image](https://github.com/SunMinghui19/k8s-hadoop-hdfs-/blob/master/image/hdfs-princple.png)
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
