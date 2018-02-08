#
#####参考：[https://tech.meituan.com/kafka-fs-design-theory.html](https://tech.meituan.com/kafka-fs-design-theory.html)

<br>
![](/assets/kafka文件存储-1.png)

同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。