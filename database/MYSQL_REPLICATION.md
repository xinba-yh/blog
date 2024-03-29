**mysql复制是什么？**

Mysql内建的复制功能是构建基于Mysql的大规模、高性能应用的基础。复制功能不仅有利于构建高性能应用，同时也是高可用性、高扩展性、灾难恢复、备份以及数据仓库等工作的基础。



**复制概述**

复制解决的基本问题是让一台服务器与其他服务器保持同步。一台主库的数据可以同步到多台备库上，备库本身也可以被配置另外一个服务器的主库。主库和备库之间可以有多种不同的组合方式。



**复制方式**

1、基于行的复制 since mysql 5.1版本

2、基于语句的复制（逻辑复制） since mysql 3.23版本

两种方式都是通过主库上记录二进制日志文件，在备库中重放日志的方式来实现异步的数据复制。异步也就意味着同一时间点备库上的数据可能与主库存在不一致，并且无法保证主备之间的延迟，几秒、几分钟甚至几小时。



**兼容性**

Mysql复制大部分是向后兼容的，新版本的服务器可以作为老版本服务器的备库，但反过来，老版本的服务器作为新版本服务器的备库通常是不行的，因为它可能无法解析新版本所采用的新的特性或语法。

PS：小版本升级，比如5.7.1 -> 5.7.2，通常是兼容的。



**性能开销**

1、复制通常不会增加主库的开销，主要是启用二进制日志带来的开销，但出于备份或及时从崩溃中恢复的目的，这点开销是必须的。

2、每个备库会对主库增加一些负载，例如网络I/O开销，尤其当备库请求从主库读取旧的二进制日志文件时可能造成更高的I/O开销。

3、如果从一个吞吐量很高（例如5000或更高的TPS）的主库上复制到多个备库，唤醒多个复制线程发送事件的开销将会增加。

4、锁竞争也可能阻碍事物的提交。



通过复制可以将读操作指向备库来获得更好的读性能，但对于写操作，除非设计得当，否则并不适合通过复制来扩展写操作。

当使用一主多备架构时，可能会造成一些浪费，因为本质上它会复制大量不必要的重复数据。备库数量，依据数据重要性自行选择。



**复制解决的问题**

<img src="all_images/image-20230214154428235.png" width=70% height=70% />

**复制如何工作？**

<img src="all_images/image-20230214153025379.png" width=70% height=70% />



**流程概述**

1、主库上把数据更改记录到二进制文件(Binary Log)中。 PS：这些记录被称为二进制日志事件。

在每次准备提交事物完成数据更新之前，主库将数据更新到事件记录到二进制日志(Binary Log)中。Mysql会按照事物提交的顺序而非每条语句的执行顺序来记录二进制日志。在完成二进制日志记录之后，主库会告诉存储引擎可以提交事物了。

2、备库将主库上的日志复制到自己的中继日志(Relay Log)中。

2.1、备库会启动一个工作线程，称为I/O线程。

2.2、备库I/O线程跟主库建立一个普通的客户端连接，然后在主库上启动一个特殊的二进制转储(binlog dump)线程，这个二进制转储线程会读取主库上二进制日志中的事件。它不会对事件进行轮询，如果该线程追上来主库，将会进行睡眠状态，直到主库发送信号量通知其有新的事件产生时才会被唤醒。

2.3、备库I/O线程会将接收到的事件记录到中继日志中。

3、备库读取中继日志中的事件，将其重放到备库数据之上。

PS：备库SQL重放线程执行的事件可以通过配置选项决定是否写入其自己的二进制日志中。



这种复制架构实现了获取事件和重放事件的解耦，允许这两个过程异步执行。日志复制I/O线程能够独立于SQL重放线程之外工作。

PS：但这种架构也限制了复制的过程，最重要的一点为主库上并发运行的查询在备库上只能串行化执行，因为只有一个SQL重放线程来重放中继日志的事件。

​      
