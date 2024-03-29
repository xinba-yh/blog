Java运行一个简单的对象创建Class class = new Class()发生了什么？

1、检查常量池中是否有类的符号引用

2、检查符号引用代表的类是否已被加载、解析、初始化。

​       2.1、没有

​	  执行类加载过程

​       2.2、有

​	  虚拟机为新对象分配内存。（Java对象所需内存大小在类加载完成后便可完全确认）



#### 对象空间分配方式

对象分配空间等同于把一块确定大小的内存从java堆中划分出来，具体的分配方式如下：

**指针碰撞（Bump The Pointer）**

假设Java堆中的内存是规整的，使用过的内存放一边，空闲的内存放另一边，中间放着一个指针作为分界点的指示器，那么分配内存就仅仅需要将指针向空闲空间移动对象大小相等的距离。

**空闲列表（Free List）**

假设Java堆中的内存是不规整的，已使用的内存和未使用的内存交错在一起，那就没有办法进行简单的指针碰撞了，JVM就需要维护一个列表，记录哪块内存块是可用的。分配的时候从可用列表中找到一块足够大空间的内存块分配给对象实例，并更新列表上的记录。

**如何选择**

具体选择哪种分配方式是由Java堆是否规整决定的，而Java堆是否规整又由所采用的垃圾收集器是否带有**空间压缩整理（Compact）**能力决定的。

1、使用Serial、ParNew等带有压缩整理过程的收集器使用指针碰撞，简单又高效。

2、使用CMS这种基于清除（Sweep）算法的收集器时，理论上只能采用空闲列表方式分配内存。



#### 并发安全

对象创建是非常频繁的行为，仅仅修改指针指向的位置是线程不安全的，而如果采用同步锁方式又会降低性能。

1、采用**CAS+失败重试**的方式保证更新操作的原子性。

2、分配动作按照线程划分在不同的空间中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB），本地缓冲区用完了，分配新的缓冲区时才需要同步锁定。

通过-XX +/-UseTLAB参数来设定是否使用TLAB。
