GC（垃圾收集）是一个早于 Java 的概念，诞生于1960年的 Lisp 语言当时就使用了内存动态分配和垃圾收集技术，经过半个世纪发展，这项技术已经发展很成熟。之所以还要去探究 GC 和内存分配，是因为当程序万一出现了内存溢出、内存泄漏等问题的时候，当垃圾收集成为系统高并发性能瓶颈的时候，我们需要有针对性的对 JVM 自动化的内存分配和垃圾回收策略实施必要的监控和调整。



要学习 GC 必须思考以下三个问题：

- 哪些内存需要回收
- 什么时候回收
- 怎么回收



## 哪些内存需要回收

通过前面的介绍，我们知道 JVM 管理的内存中，线程私有的虚拟机栈、本地方法栈以及程序计数器都是随着线程而生，随线程而灭；栈中的栈帧随着方法的进入和退出有条不紊的进行入栈和出栈。这部分内存区域随着线程结束或者方法退出自然的就被释放回收了，因此这部分不需要过多考虑回收问题。而 Java 堆及方法区则不一样，这部分内存的分配和回收都是动态的。垃圾收集器关注的区域主要指的是这部分内存。

### 判断对象是否存活的方法

GC  在垃圾回收的时候首先需要判断哪些对象时“存活”的，哪些是已经“死亡”了（不能够再被使用到的对象）可以回收，判断的方法主要有 **引用计数法** 及 **可达性分析法**。

**引用计数法：** 给对象添加一个引用计数器，每当有一个地方引用计数器就加1，引用失效时计数器就减1，因此哪些计数器为0的对象都是不再被引用需要回收的对象。优点是实现简单、效率也高，但是缺点就是无法解决对象之间相互引用的情况。

**可达性分析法（HotSpot 默认）：** 通过一系列称为 **GC Roots** 的对象作为起点开始向下进行搜索，搜索走过的路径叫做引用链，如果一个对象没有任何引用链与其相连时说明该对象不可达，即不可能再被使用到，这时便被判定为可回收的对象。在 Java 中 **可作为 GC Roots 的对象**有以下几种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 本地方法栈中 Native 方法引用的对象
- 方法区中的常量引用的对象

需要注意的一点是在可达性分析方中不可达的对象也并非就一定是“非死不可”的，GC 在第一次可达性分析发现对象不可达时进行一次标记，同时会判断该对象是否有必要执行 **finalize()** 方法（当对象覆盖了finalize()方法并且没有被调用过时才会被认为有必要），如果判断有必要则将其加入一个叫作 **F-Queue** 的队列，稍后虚拟机会启动一个专门的、低优先级的线程去依次执行队列中对象的 finalize 方法，并且不保证一定执行成功。在对象的 finalize 方法中如果对象将自己（this关键字）与引用链上的任意对象建立关联，那么在下一次可达性标记的时候这个对象就会被从“即将回收”的集合中移除。代码举例见p67。注意 **finalize()** 方法不被建议在代码中使用，因为其能做的事情 **try-finally** 都能够做的更好！



## 垃圾怎么回收（垃圾回收算法）

### 标记-清除算法（Mark-Sweep）

这时最基础的收集算法，分为“标记”和“清除”两个阶段：首先按照上面介绍的方法标记出需要回收的对象，标记完成后统一对被标记的对象内存进行回收。显然这种方式会导致产生大量不连续的内存碎片，从而导致后面再需要分配较大对象时无法找到足够的连续内存，从而提前触发另一次垃圾收集。

### 复制算法（Coping）

这种算法将可用内存按照容量划分为大小相等的两部分，每次只使用其中一半，当这一半使用完了就将其中还存活的对象复制到另一块内存上，然后对这一块内存进行回收，循环往复。优点是实现简单、运行高效。缺点就是可用内存缩小到了原来的一半，这个代价稍微有点高！

这种算法主要被用来回收新生代，因为新生代中的对象百分之九十八都是“朝生夕死”，也就是说大部分内存都会被回收掉，那就没有必要按照1:1的比例划分内存空间，而是将内存分为较大的一块 Eden 空间和两块较小的 Survivor 空间。每次使用 Eden 和其中一块 Survivor 区（from），当回收时将其中的存活对象复制到另外一块 Survivor 区，把 Eden 和刚才用过的 Survivor 区清理掉。HotSpot 虚拟机默认的 Eden 和 Survivor 内存比例是 **8:1**，也就是说每次新生代的可用空间为整个新生代容量的90%，这样内存的利用率很高，一定程度上避免了上面提到的可用内存折半的缺点。但是我们并没有办法保证每次回收都只有不到10%的对象存活（因为存活的对象会被复制到 survivor to 区，这部分只占了10%），这样就有可能出现 Survivor 内存不够用，需要依赖其它内存（老年代）进行分配担保。

显然在对象存活率较高的情况下这种算法效率就会降低。

### 标记整理算法（Mark-Compact）

老年代由于存活率比较高（想想为什么），因此并不适合上面提到的复制算法，针对其特点，“标记-整理”的算法被提出来。其标记过程与“标记-清除”算法的过程一样，但后续并不是直接对标记对象进行清理，而是让存活的对象都向一端移动，然后直接清理掉边界以外的区域。

当代虚拟机都采用“分代收集”的思想，一般根据对象存活周期将 Java 堆分为新生代和老年代，分别根据其特点选择相应的收集算法：新生代对象存活率低，则采用**复制算法**只需要对极少比例的存活对象进行复制即可完成收集；而老年代因为存活率高，没有额外空间对其进行分配担保，就必须使用**“标记-清理”或者“标记-整理”算法**来回收。



## 如何高效的在堆上分配对象空间

java在堆上分配对象，在堆栈上分配指向对象的引用及常量。java从堆分配空间的速度可以和其它程序在堆栈上分配分配空间的速度相媲美。只是因为在分配堆空间时java只是简单的将“堆指针”移动到尚未分配的区域，他是如何做到的呢？
这得益于java的垃圾回收器的工作原理，gc在工作的时候一边回收空间，一边将堆中的对象重新紧凑的排列，这样“堆指针”就可以更容易的移到空闲区域的开始处，也就尽量避免了页面错误。通过gc对对象重新排列，实现了一种高速的、有无限空间可供分配的堆模型。

## 垃圾回收的思想

> 对任何“活”的对象，一定能最终追溯到其存活在堆栈或静态存储区中的引用。

基于此从堆栈和静态存储区开始遍历所有的引用，就能找到所有“活”的对象，对这些对象进行标记，将其余的对象回收。不同的虚拟机对这个过程有不同的具体实现。

1. 停止-复制模式
先暂停程序运行（不属于后台回收模式），将所有活得对象从当前堆复制到另一个堆，没有复制的对象都当作垃圾回收，复制到新堆时对象会被一个挨着一个整齐的排列，这样便可以按照前面说的移动“堆指针”的方式直接分配新空间了。当然这种“复制移动”式的回收方法效率较低，通常做法是按需从堆中分配几块较大的内存，复制动作发生在这几块较大的内存之间。

2. 标记-清扫模式
前一种“停止-复制”模式在垃圾较少的情况下效率仍然很低下，因为这时大量的复制行为其实没有必要，于是另一种新的方法：遍历所有引用进而找到所有存活的对象并对其标记，标记完成以后将没有标记的对象清理，这个过程中并不做任何复制。当然这样的话剩下的堆空间并不是连续的。

两种方式个有利弊，一般java虚拟机会采用一种自适应的方式，即如果监控到所有对象都很稳定垃圾回收器效率较低时，切换到“标记-清扫模式”，同样监控发现堆空间出现很多碎片时又切回“停止-复制”模式。