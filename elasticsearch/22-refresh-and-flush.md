##### 参考文档：[https://qbox.io/blog/refresh-flush-operations-elasticsearch-guide](https://qbox.io/blog/refresh-flush-operations-elasticsearch-guide)

##Refresh and Flush
&nbsp;&nbsp;&nbsp;&nbsp;初看，Refresh和Flush操作的目的似乎是相同的。二者都用于使文档可被搜索。当ES中加入一个新document的时候，我们使用Refresh或Flush操作来使得其可被索引。为了更好的理解，我们需要熟悉ES的底层引擎Lucene中Segments、Reopen和Commits的概念。
##Lucene中的Segments
&nbsp;&nbsp;&nbsp;&nbsp;ES中，数据存储单元叫做shard。但是这从Lucene的角度来看的话，又有些不同。一个ES的shard在Lucene中是一个索引，而且每一个索引又包含许多的Lucene Segments。一个Segment是包含document以及其包含的terms的倒排索引。
&nbsp;&nbsp;&nbsp;&nbsp;Segment的概念以及其在ES的index和shard中的应用如下图所示：  
![](/assets/lucene1.png)  
&nbsp;&nbsp;&nbsp;&nbsp;这种分段设计背后的概念是：每当创建新的document的时候，他们都会被写入新的segment中。无论新的文档是什么时候被创建，它都会被写入新的segment中，而不会去修改原先的segments。如果一个document要被删除，则在原始段中将其标记为deleted。这意味着它并不会从段中被物理的删除。
##Lucene Reopen
&nbsp;&nbsp;&nbsp;&nbsp;当调用Lucene Reopen操作后，将使得积累的数据可被搜索。虽然最新的数据可以用于搜索，但这并不能保证数据的持久性，或者未被写入磁盘。我们可以重复调用该操作N次，使最新数据可搜索，但不能确保数据已经被写入磁盘中。
##Lucene中的Commits
&nbsp;&nbsp;&nbsp;&nbsp;Commits操作使得数据变得安全。对于每次Commits操作，将会使来自不同段的数据合并并推入磁盘，使数据持久化。尽管Commits是保证数据状态的理想方式，但问题是每次Commits操作都是耗费资源的。每次Commits操作都有自己的I/O操作和与之相关联的读/写周期。这导致昂贵的操作。这就是为什么我们更喜欢在基于Lucene的系统中启用Reopen功能以使新数据可搜索的原因。  
##Translog
&nbsp;&nbsp;&nbsp;&nbsp;ES在持久性的问题上采用了一种不同的方法。它在每一个shard中引入了translog (transaction log)。可被索引的documents会被传递到translog和内存缓冲区中，这个过程如下图所示：  
![](/assets/lucene2.png)  
##Elasticsearch中的Refresh
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下，每秒钟执行一次Refresh操作。Refresh操作中，将内存缓冲区中的内容清空到内存中新创建的Segment，其如下图所示。这使得新数据可用于搜索。  
![](/assets/lucene3.png)  
##Translog和持久性
&nbsp;&nbsp;&nbsp;&nbsp;针对持久性这个问题，translog是如何工作的呢？translog存在于每一个shard中，这意味着它属于物理磁盘存储器。即使发生错误，translog中的数据也会在shard水平上得以保留。当节点重启时，ES能够从translog中恢复数据。
&nbsp;&nbsp;&nbsp;&nbsp;translog在每隔一个设定的时间间隔或者成功完成一个请求（Index、Bulk、Delete或者Update）时被提交到磁盘。
##Elasticsearch中的Flush
&nbsp;&nbsp;&nbsp;&nbsp;如图三所示，Flush实质上是指内存缓冲区中的所有文档被写入新Segment。然后如图四所示，所有的现有的内存中的Segments被提交到磁盘，并且会清空translog。  
![](/assets/lucene4.png)    
&nbsp;&nbsp;&nbsp;&nbsp;Flush操作被周期性或者当translog达到特定大小时触发。这些设置是为了防止Lucene commits操作的不可控开销。
##结论
&nbsp;&nbsp;&nbsp;&nbsp;在本指南中，我们以可理解的方式探讨了两种密切相关的操作：Refresh和Flush。我们还介绍了Lucene功能的基础知识，如Reopen和Commits，这有助于理解Refresh和Flush操作。  
&nbsp;&nbsp;&nbsp;&nbsp;简言之，Refresh用于使新的document可被搜索。Flush是通过磁盘来保证内存中Segments的持久性。Flush不影响documents在ES中的可见性，因为搜索发生在内存中的Segments。Refresh影响索引文档的可见性。