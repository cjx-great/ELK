### **一个Broker:**

######参考：[https://tech.meituan.com/kafka-fs-design-theory.html](https://tech.meituan.com/kafka-fs-design-theory.html)

<br>
![](/assets/kafka文件存储-1.png)

* 同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。
* 一个partition目录下分为多个大小相等的segment文件（由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件）。其命名规则：文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。segment大小相同，但是里面存储的消息数目可能不同。  

<br>
![](/assets/index-log关系.png)

* 索引采取稀疏索引存储方式，它减少了索引文件大小。稀疏索引为数据文件的每个对应message设置一个元数据指针,它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间。
* 在partition中通过offset查找message：
1. 定位到index/log文件。
2. 查找索引文件确定偏移量，然后根据该偏移量到数据文件中顺序查找。



### **多个Broker:**
######参考：[http://blog.csdn.net/lizhitao/article/details/41778193](http://blog.csdn.net/lizhitao/article/details/41778193)
假如：集群中由4个Broker，1个topic包含4个Partition，2个Replication，则：  
![](/assets/Kafka集群partitions_replicas分配.png)

现在增加两个Broker,增加两个partition,则：  
![](/assets/增加broker-partition.png)

* 副本分配规则：<br>
  每个Broker(按照BrokerId有序)依次分配主Partition,下一个Broker为副本，如此循环迭代分配，多副本都遵循此规则。
* 副本分配算法:<br>
 1. 将所有N个Broker和待分配的i个Partition排序
 2. 将第i个Partition分配到第(i mod n)个Broker上
 3. 将第i个Partition的第j个副本分配到第((i + j) mod n)个Broker上