### 什么是JVM

JVM 是可运行 Java 代码的假想计算机 ，包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收，堆 和 一个存储方法域。JVM 是运行在操作系统之上的，它与硬件没有直接的交互。

![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnfnBgKtQhKt3eSFrhgLxiaJweD0771O8LIKHdO6urvB9k7oT70RF6uAA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述


我们都知道 Java 源文件，通过编译器，能够生产相应的.Class 文件，也就是字节码文件，而字节码文件又通过 Java 虚拟机中的解释器，编译成特定机器上的机器码 。

![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnrk3ybrDuRmG4lFM7vibnoia5azUvO11sN9Lp765yibNVY4MpU6jl5GXaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述


每一种平台的解释器是不同的，但是实现的虚拟机是相同的，这也就是 Java 为什么能够跨平台的原因了 ，当一个程序从开始运行，这时虚拟机就开始实例化了，多个程序启动就会存在多个虚拟机实例。程序退出或者关闭，则虚拟机实例消亡，多个虚拟机实例之间数据不能共享。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnuEnRIrlriavg62M4baO7eGwUCS1ibLgpI6kJr5xyw3v5Jp3iazJowD06w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###### 线程

这里所说的线程指程序执行过程中的一个线程实体。JVM 允许一个应用并发执行多个线程 。Hotspot JVM 中的 Java 线程与原生操作系统线程有直接的映射关系。当线程本地存储、缓冲区分配、同步对象、栈、程序计数器等准备好以后，就会创建一个操作系统原生线程。Java 线程结束，原生线程随之被回收。操作系统负责调度所有线程，并把它们分配到任何可用的 CPU 上。当原生线程初始化完毕，就会调用 Java 线程的 run() 方法。当线程结束时，会释放原生线程和 Java 线程的所有资源。

- Hotspot JVM 后台运行的系统线程主要有下面几个：
- **虚拟机线程**:这个线程等待 JVM 到达安全点操作出现。这些操作必须要在独立的线程里执行，因为当堆修改无法进行时，线程都需要  JVM位于安全点。这些操作的类型有：stop-the-world 垃圾回收、线程栈dump、线程暂停、线程偏向锁（biased  locking）解除。
- **周期性任务线程**:这线程负责定时器事件（也就是中断），用来调度周期性操作的执行。
- **GC 线程**  :这些线程支持 JVM 中不同的垃圾回收活动。
- **编译器线程**:这些线程在运行时将字节码动态编译成本地平台相关的机器码。
- **信号分发线程**:这个线程接收发送到 JVM 的信号并调用适当的 JVM 方法处理。

### JVM内存区域

![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnLe1qertTmVahQ6GWmJvjurte6FOgIkrTypzalyiccjpvFIvEhBfZQiaw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

- JVM 内存区域主要分为线程私有区域【**程序计数器、虚拟机栈、本地方法区**】、线程共享区域【**JAVA 堆、方法区**】、直接内存。
  -**线程私有数据区域生命周期与线程相同**, 依赖用户线程的启动/结束 而 创建/销毁(在 HotspotVM 内, 每个线程都与操作系统的本地线程直接映射, 因此这部分内存区域的存/否跟随本地线程的生/死对应)。
- **线程共享区域**随虚拟机的启动/关闭而创建/销毁。

![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnm4Bic2rtLxUGpq66DqstrwEkzp6fS9HrkbH5pu4ba2w1nZGujzUvZPg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

###### 程序计数器( 线程私有）

- 一块较小的内存空间, 是当前线程所执行的字节码的行号指示器，每条线程都要有一个独立的程序计数器，这类内存也称为“线程私有”的内存。
- 正在执行 java 方法的话，计数器记录的是虚拟机字节码指令的地址（当前指令的地址）。如果还是 Native 方法，则为空。
- 这个内存区域是唯一一个在虚拟机中没有规定任OutOfMemoryError 情况的区域。

###### JAVA虚拟机栈( 线程私有)

- **是描述java方法执行的内存模型，每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息**。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

- 栈帧（ Frame）是用来存储数据和部分过程结果的数据结构，同时也被用来处理动态链接(Dynamic Linking)、 方法返回值和异常分派（ Dispatch Exception）。**栈帧随着方法调用而创建，随着方法结束而销毁**——无论方法是正常完成还是异常完成（抛出了在方法内未被捕获的异常）都算作方法结束。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnEZ6iaGeKEBO3iayFic5TV3naQwJV7gIBTEMRQrzXibia1kU2NXajU0ic9VIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

###### 本地方法区(线程私有）

- 本地方法区和 **Java Stack** 作用类似, 区别是虚拟机栈为执行 Java 方法服务, 而**本地方法栈则为Native 方法服务**, 如果一个 VM 实现使用 C-linkage 模型来支持 Native 调用, 那么该栈将会是一个C 栈，但 HotSpot VM 直接就把本地方法栈和虚拟机栈合二为一。

###### 堆（Heap- 线程共享）运行时数据区

- 是被线程共享的一块内存区域，创建的对象和数组都保存在 Java 堆内存中，也是垃圾收集器进行垃圾收集的最重要的内存区域。由于现代 VM 采用分代收集算法, 因此 Java 堆从 GC 的角度还可以细分为: **新生代( Eden 区 、 From Survivor 区 和 To Survivor 区 )和老年代**(jdk1.7)。

###### 方法区/ 永久代 （线程共享）

- 即我们常说的永久代(Permanent Generation), 用于存储被 **JVM 加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**. HotSpot VM把GC分代收集扩展至方法区, 即使用Java堆的永久代来实现方法区, 这样 HotSpot 的垃圾收集器就可以像管理  Java 堆一样管理这部分内存,而不必为方法区开发专门的内存管理器(永久带的内存回收的主要目标是针对常量池的回收和类型的卸载,  因此收益一般很小)。

###### 运行时常量池

- （Runtime Constant Pool）是方法区的一部分。Class  文件中除了有类的版本、字段、方法、接口等描述等信息外，还有一项信息是常量池（Constant Pool  Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。Java 虚拟机对 Class  文件的每一部分（自然也包括常量池）的格式都有严格的规定，每一个字节用于存储哪种数据都必须符合规范上的要求，这样才会被虚拟机认可、装载和执行。

###### 直接内存

- 直接内存并不是 JVM 运行时数据区的一部分, 但也会被频繁的使用: 在 JDK 1.4 引入的 NIO 提供了基于 Channel 与 Buffer 的  IO 方式, 它可以使用 Native 函数库直接分配堆外内存, 然后使用DirectByteBuffer 对象作为这块内存的引用进行操作，  这样就避免了在 Java堆和 Native 堆中来回复制数据, 因此在一些场景中可以显著提高性能。

### JVM运行时内存(jdk1.7)

- Java 堆从 GC 的角度还可以细分为: **新生代**( Eden 区 、 From Survivor 区 和 To Survivor 区 )和**老年代**

  ![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadn63GZYLA8Gf1fDQE4ia5AmxzA3jONYP7PPB7Cbcl6OvdbGpGHyeER1aA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

###### 新生代

是用来存放新生的对象。一般占据堆的1/3空间。由于频繁创建对象，所以新生代会频繁触发MinorGC 进行垃圾回收。新生代又分为 Eden 区、ServivorFrom、ServivorTo 三个区。

- Eden区：Java新对象的出生地（如果新创建的对象占用内存很大，则直接分配到老年代）。当Eden区内存不够的时候就会触发MinorGC，对新生代区进行一次垃圾回收。
- ServivorFrom：上一次 GC 的幸存者，作为这一次 GC 的被扫描者。
- ServivorTo：保留了一次 MinorGC 过程中的幸存者。
- MinorGC 的过程：（复制->清空->互换）MinorGC 采用复制算法。
- **eden 、 servicorFrom  复制到 ServicorTo，年龄+1**
     首先，把 Eden和 ServivorFrom区域中存活的对象复制到  ServicorTo区域（如果有对象的年龄以及达到了老年的(默认15岁，可以通过-XXMaxTenuringThreshold设置)，则赋值到老年代区），同时把这些对象的年龄+1（如果 ServicorTo 不够位置了就放到老年区）。
- **清空 eden 、 servicorFrom****
     清空 Eden 和 ServicorFrom 中的对象；
- **ServicorTo 和 ServicorFrom 互换**
     最后，ServicorTo 和 ServicorFrom 互换，原 ServicorTo 成为下一次 GC 时的 ServicorFrom区。

###### 老年代

- 主要存放应用程序中生命周期长的内存对象。
- 老年代的对象比较稳定，所以 MajorGC 不会频繁执行。在进行 MajorGC 前一般都先进行了一次  MinorGC，使得有新生代的对象晋身入老年代，导致空间不够用时才触发。当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次  MajorGC 进行垃圾回收腾出空间。
- MajorGC 采用**标记清除算法**：首先扫描一次所有老年代，标记出存活的对象，然后回收没有标记的对象。MajorGC 的耗时比较长，因为要扫描再回收。MajorGC  会产生内存碎片，为了减少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。当老年代也满了装不下的时候，就会抛出 OOM（Out of  Memory）异常。

###### 永久代

-指内存的永久保存区域，主要存放 Class 和 Meta（元数据）的信息,Class 在被加载的时候被放入永久区域，它和和存放实例的区域不同,**GC 不会在主程序运行期对永久区域进行清理**。所以这也导致了永久代的区域会随着加载的 Class 的增多而胀满，最终抛出 OOM 异常。

###### JAVA8 与元数据

在Java8中，**永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代**。元空间的本质和永久代类似，元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制**。类的元数据放入 nativememory, 字符串池和类的静态变量放入 java 堆中，这样可以加载多少类的元数据就不再由MaxPermSize 控制, 而由系统的实际可用空间来控制。

### 垃圾回收与算法

![图片](https://mmbiz.qpic.cn/mmbiz_png/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnGBFGL7wxXFMNjuj1StictFMmtUW3Z0QicvEnQc4Jib1Qg21X1MwhIaJSQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

#### 如何确定垃圾

###### 引用计数法

- 在 Java  中，引用和对象是有关联的。如果要操作对象则必须用引用进行。因此，很显然一个简单的办法是通过引用计数来判断一个对象是否可以回收。简单说，即一个对象如果没有任何与之关联的引用，即他们的引用计数都不为 0，则说明对象不太可能再被用到，那么这个对象就是可回收对象。

###### 可达性分析

- 为了解决引用计数法的循环引用问题，Java 使用了可达性分析的方法。通过一系列的“GC roots”对象作为起点搜索**。如果在“GC roots”和一个对象之间没有可达路径，则称该对象是不可达的**。要注意的是，不可达对象不等价于可回收对象，**不可达对象变为可回收对象至少要经过两次标记过程**。两次标记后仍然是可回收对象，则将面临回收。

#### 标记清除算法（ Mark-Sweep ）

1. 最基础的垃圾回收算法，分为两个阶段，标注和清除。标记阶段标记出所有需要回收的对象，清除阶段回收被标记的对象所占用的空间。如图

   ![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)在这里插入图片描述

   
   从图中我们就可以发现，该算法最大的问题是内存碎片化严重，后续可能发生大对象不能找到可利用空间的问题。

#### 复制算法（copying ）

- 为了解决 Mark-Sweep 算法内存碎片化的缺陷而被提出的算法。按内存容量将内存划分为等大小的两块。每次只使用其中一块，当这一块内存满后将尚存活的对象复制到另一块上去，把已使用的内存清掉，如图：

  ![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnHYw9oWWruwgD1TfSeZfM5nS803T3vamSxWhyqjicmZibam6cFMBV3dtA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

  
  这种算法虽然实现简单，内存效率高，不易产生碎片，但是最大的问题是可用内存被压缩到了原本的一半。且存活对象增多的话，Copying 算法的效率会大大降低。

#### 标记整理算法(Mark-Compact)

结合了以上两个算法，为了避免缺陷而提出。标记阶段和 Mark-Sweep 算法相同，标**记后不是清理对象，而是将存活对象移向内存的一端。然后清除端边界外的对象**。如图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadn5fajONYEQPlTepljzkWsibg39mROdtnRiaibAhRrGjoGB2NQkjkxy16xA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

### 分代收集算法

- 分代收集法是目前大部分 JVM 所采用的方法，其核心思想是根据对象存活的不同生命周期将内存划分为不同的域，一般情况下将 GC 堆划分为老生代(Tenured/Old  Generation)和新生(YoungGeneration)。老生代的特点是每次垃圾回收时只有少量对象需要被回收，新生代的特点是每次垃圾回收时都有大量垃圾需要被回收，因此可以根据不同区域选择不同的算法。

##### 新生代与复制算法

- 目前大部分 JVM 的 GC 对于新生代都采取 Copying 算法，因为新生代中每次垃圾回收都要回收大部分对象，即要复制的操作比较少，但通常并不是按照  1：1 来划分新生代。一般将新生代划分为一块较大的 Eden 空间和两个较小的 Survivor 空间(From Space, To  Space)，每次使用Eden 空间和其中的一块 Survivor 空间，当进行回收时，将该两块空间中还存活的对象复制到另一块 Survivor 空间中。

  ![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadndAvGtsj62X8kjsdHJFAKkKJvl6uOMjvtic57UyWejQ3oEE7HJqtSbFg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

##### 老年代与标记复制算法

而老年代因为每次只回收少量对象，因而采用 Mark-Compact 算法。

- JAVA 虚拟机提到过的处于**方法区的永生代(Permanet Generation)，它用来存储 class 类，常量，方法描述等**。对永生代的回收主要包括废弃常量和无用的类。
- 对象的内存分配主要在新生代的 Eden Space 和 Survivor Space 的 From Space(Survivor 目前存放对象的那一块)，少数情况会直接分配到老生代。
- 当新生代的 Eden Space 和 From Space 空间不足时就会发生一次 GC，进行 GC 后，EdenSpace 和 From Space 区的存活对象会被挪到 To Space，然后将 Eden Space 和 FromSpace 进行清理。
- 如果 To Space 无法足够存储某个对象，则将这个对象存储到老生代。
- 在进行 GC 后，使用的便是 Eden Space 和 To Space 了，如此反复循环。
- 当对象在 Survivor 区躲过一次 GC 后，其年龄就会+1。默认情况下年龄到达 15 的对象会被移到老生代中。

### GC  分代收集算法 VS  分区收集算法

#### 分代收集算法

当前主流 JVM 垃圾收集都采用”分代收集”(Generational Collection)算法, 这种算法会根据对象存活周期的不同将内存划分为几块, 如 JVM 中的 **新生代、老年代、永久代**，这样就可以根据各年代特点分别采用最适当的 GC 算法。

- **在新生代-复制算法**    
  每次垃圾收集都能发现大批对象已死, 只有少量存活. 因此选用复制算法, 只需要付出少量存活对象的复制成本就可以完成收集。
- **在老年代-标记整理算法**
  因为对象存活率高、没有额外空间对它进行分配担保, 就必须采用“**标记—清理”或“标记—整理”**算法来进行回收, 不必进行内存复制, 且直接腾出空闲内存。

#### 分区收集算法

**分区算法则将整个堆空间划分为连续的不同小区间, 每个小区间独立使用, 独立回收. 这样做的好处是可以控制一次回收多少个小区间** , 根据目标停顿时间, 每次合理地回收若干个小区间(而不是整个堆), 从而减少一次 GC 所产生的停顿。

### GC 垃圾收集器

Java 堆内存被划分为新生代和年老代两部分，新生代主要使用复制和标记-清除垃圾回收 算法 ,年老代主要使用标记-整理垃圾回收算法，因此 java  虚拟中针对新生代和年老代分别提供了多种不同的垃圾收集器，JDK1.6 中 Sun HotSpot 虚拟机的垃圾收集器如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)![img](https://oscimg.oschina.net/oscnet/840b196dc3e9c3b432251a5eb96cbd7c8e9.jpg)

#### Serial  垃圾收集器 （单线程、 复制算法 ）

- **Serial（英文连续）是最基本垃圾收集器，使用复制算法**，曾经是JDK1.3.1之前新生代唯一的垃圾收集器。Serial 是一个单线程的收集器，它不但只会使用一个 CPU  或一条线程去完成垃圾收集工作，并且在进行垃圾收集的同时，必须暂停其他所有的工作线程，直到垃圾收集结束。Serial  垃圾收集器虽然在收集垃圾过程中需要暂停所有其他的工作线程，但是它简单高效，对于限定单个 CPU  环境来说，没有线程交互的开销，可以获得最高的单线程垃圾收集效率，因此 **Serial垃圾收集器依然是 java 虚拟机运行在 Client 模式下默认的新生代垃圾收集器。**

#### ParNew  垃圾收集器 （Serial+ 多线程 ）

- **ParNew 垃圾收集器其实是 Serial 收集器的多线程版本，也使用复制算法**，除了使用多线程进行垃圾收集之外，其余的行为和 Serial 收集器完全一样，ParNew 垃圾收集器在垃圾收集过程中同样也要暂停所有其他的工作线程。ParNew 收集器默认开启和 CPU  数目相同的线程数，可以通过-XX:ParallelGCThreads  参数来限制垃圾收集器的线程数。【Parallel：平行的】ParNew虽然是除了多线程外和Serial收集器几乎完全一样，**但是ParNew垃圾收集器是很多java虚拟机运行在 Server 模式下新生代的默认垃圾收集器。**

#### Parallel Scavenge  收集器 （多线程复制算法、高效）

- Parallel Scavenge 收集器也是一个新生代垃圾收集器，同样使用复制算法，也是一个多线程的垃圾收集器，**它重点关注的是程序达到一个可控制的吞吐量**（Thoughput，CPU 用于运行用户代码的时间/CPU 总消耗时间，即吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间)），高吞吐量可以最高效率地利用 CPU 时间，尽快地完成程序的运算任务，主要适用于在后台运算而不需要太多交互的任务。**自适应调节策略也是 ParallelScavenge 收集器与 ParNew 收集器的一个重要区别。**

#### Serial Old  收集器 (单线程标记整理算法）

Serial Old 是 Serial 垃圾收集器年老代版本，它同样是个单线程的收集器，使用标记-整理算法，这个收集器也主要是运行在 Client 默认的 java 虚拟机默认的年老代垃圾收集器。在 Server 模式下，主要有两个用途：
   \1. 在 JDK1.5 之前版本中与新生代的 Parallel Scavenge 收集器搭配使用。
   \2. 作为年老代中使用 CMS 收集器的后备垃圾收集方案。
新生代 Serial 与年老代 Serial Old 搭配垃圾收集过程图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadn8aykozMaCrgTqHBvPb8Gric3TQ96wcGR9lYma44DmLpMHYEgKFbqTSg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述


新生代 Parallel Scavenge 收集器与 ParNew  收集器工作原理类似，都是多线程的收集器，都使用的是复制算法，在垃圾收集过程中都需要暂停所有的工作线程。新生代  ParallelScavenge/ParNew 与年老代 Serial Old 搭配垃圾收集过程图：         

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnMnVArWcj9LU5FoEzLhtYoqElQSW06m5pPicdZtaicFf90JjwWuJbibyeg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

#### Parallel Old  收集器（多线程标记整理算法）

Parallel Old收集器是Parallel Scavenge的年老代版本，使用多线程的标记-整理算法，在JDK1.6才开始提供。在 JDK1.6  之前，新生代使用 ParallelScavenge 收集器只能搭配年老代的 Serial Old  收集器，只能保证新生代的吞吐量优先，无法保证整体的吞吐量，Parallel Old  正是为了在年老代同样提供吞吐量优先的垃圾收集器，如果系统对吞吐量要求比较高，可以优先考虑新生代 Parallel Scavenge和年老代  Parallel Old 收集器的搭配策略。新生代 Parallel Scavenge 和年老代 Parallel Old  收集器搭配运行过程图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)在这里插入图片描述

#### CMS  收集器 （多线程标记清除算法）

Concurrent mark  sweep(CMS)收集器是一种年老代垃圾收集器，其最主要目标是获取最短垃圾回收停顿时间，和其他年老代使用标记-整理算法不同，它使用多线程的标记-清除算法。最短的垃圾收集停顿时间可以为交互比较高的程序提高用户体验。CMS 工作机制相比其他的垃圾收集器来说更复杂，整个过程分为以下 4 个阶段：
  **1.初始标记**：只是标记一下 GC Roots 能直接关联的对象，速度很快，仍然需要暂停所有的工作线程。
  **2.并发标记：** 进行 GC Roots 跟踪的过程，和用户线程一起工作，不需要暂停工作线程。
  **3.重新标记：** 为了修正在并发标记期间，因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，仍然需要暂停所有的工作线程。
  **4.并发清除：** 清除 GC Roots 不可达对象，和用户线程一起工作，不需要暂停工作线程。由于耗时最长的并
发标记和并发清除过程中，垃圾收集线程可以和用户现在一起并发工作，所以总体上来看**CMS 收集器的内存回收和用户线程是一起并发地执行。**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/qu3ItokgsAq4fwdWI7g9ibibBcKOV1eadnmlWcWbBL7tsusicnsZlMsvIKMFOow9QMoDbQGtL8YQ7ZZdaibQKicMib9w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述



其中，`初始标记`、`重新标记`这两个步骤仍然需要Stop-the-world。**初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC Roots  Tracing的过程，而重新标记阶段则是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始阶段稍长一些，但远比并发标记的时间短。**

> CMS以流水线方式拆分了收集周期，将耗时长的操作单元保持与应用线程并发执行。只将那些必需STW才能执行的操作单元单独拎出来，控制这些单元在恰当的时机运行，并能保证仅需短暂的时间就可以完成。这样，在整个收集周期内，只有**两次短暂的暂停（初始标记和重新标记）**，**达到了近似并发的目的**。

CMS收集器**优点**：并发收集、低停顿。

CMS收集器**缺点**：

•CMS收集器对CPU资源非常敏感。•CMS收集器无法处理浮动垃圾（Floating Garbage）。•CMS收集器是基于标记-清除算法，该算法的缺点都有（内存碎片）。•**停顿时间是不可预期的**。

CMS收集器之所以能够做到并发，根本原因在于**采用基于“标记-清除”的算法并对算法过程进行了细粒度的分解**。前面篇章介绍过标记-清除算法将产生大量的内存碎片这对新生代来说是难以接受的，因此新生代的收集器并未提供CMS版本。

> 备注：说CMS是老年代收集器，其实不是非常准确。CMS 的各个收集过程其实是一个涉及年轻代和老年代的综合性垃圾回收器，在很多文章和书籍的划分中，都将 CMS 划分为了老年代垃圾回收器，加上它主要作用于老年代，所以一般误认为是。

另外要补充一点，JVM在暂停的时候，需要选准一个时机。由于JVM系统运行期间的复杂性，不可能做到随时暂停，因此引入了安全点的概念。

**并行收集器设置**

```java
    -XX:ParallelGCThreads=n:设置并行收集器收集时最大线程数使用的 CPU 数。并行收集线程数。 

    -XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间，单位毫秒。可以减少 STW 时间。 

    -XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为 1/(1+n)并发收集器设置 

    -XX:+CMSIncrementalMode:设置为增量模式。适用于单 CPU 情况。 

    -XX:+UseAdaptiveSizePolicy：设置此选项后，并行收集器会自动选择年轻代区大小和相应的 Survivor 区比例，以达到目标系统规定的最低相应时间或者收集频率等，此值建议使用并行收集器时，一直打开。 

    -XX:CMSFullGCsBeforeCompaction=n：由于并发收集器不对内存空间进行压缩、整理，所以运行一段时间以后会产生“碎片”，使得运行效率降低。此值设置运行多少次 GC 以后对内存空间进行压缩、整理。 

    -XX:+UseCMSCompactAtFullCollection：打开对年老代的压缩。可能会影响性能，但是可以消除碎片
```

### 安全点(Safepoint)

**安全点，即程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停**。Safepoint的选定既不能太少以至于让GC等待时间太长，也不能过于频繁以致于过分增大运行时的负荷。

安全点的初始目的并不是让其他线程停下，而是找到一个稳定的执行状态。在这个执行状态下，Java虚拟机的堆栈不会发生变化。这么一来，垃圾回收器便能够“安全”地执行可达性分析。只要不离开这个安全点，Java虚拟机便能够在垃圾回收的同时，继续运行这段本地代码。

程序运行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。安全点的选定基本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的。“**长时间执行**”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生Safepoint。

对于安全点，另一个需要考虑的问题就是如何在GC发生时让所有线程（这里不包括执行JNI调用的线程）都“跑”到最近的安全点上再停顿下来。

两种解决方案：

•抢先式中断（Preemptive Suspension）

抢先式中断不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点上，就恢复线程，让它“跑”到安全点上。现在几乎没有虚拟机采用这种方式来暂停线程从而响应GC事件。

•主动式中断（Voluntary Suspension）

主动式中断的思想是当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。

### 安全区域

指在一段代码片段中，引用关系不会发生变化。在这个区域中任意地方开始GC都是安全的。也可以把Safe Region看作是被扩展了的Safepoint。

#### G1  收集器

Garbage first 垃圾收集器是目前垃圾收集器理论发展的最前沿成果，相比与 CMS 收集器，G1 收
集器两个最突出的改进是：

1. 基于标记-整理算法，不产生内存碎片。
2. 可以非常精确控制停顿时间，在不牺牲吞吐量前提下，实现低停顿垃圾回收。
   **G1 收集器避免全区域垃圾收集，它把堆内存划分为大小固定的几个独立区域**，并且跟踪这些区域的垃圾收集进度，同时在后台维护一个优先级列表，每次根据所允许的收集时间，**优先回收垃圾最多的区域**。区域划分和优先级区域回收机制，确保 G1 收集器可以在有限时间获得最高的垃圾收集效率。

G1重新定义了堆空间，打破了原有的分代模型，将堆划分为一个个区域。这么做的目的是在进行收集时不必在全堆范围内进行，这是它最显著的特点。区域划分的好处就是带来了`停顿时间可预测`的收集模型：用户可以指定收集操作在多长时间内完成。即G1提供了接近实时的收集特性。G1 的主要关注点在于达到可控的停顿时间，在这个基础上尽可能提高吞吐量。

G1 使用了停顿预测模型来满足用户指定的停顿时间目标，并基于目标来选择进行垃圾回收的区块数量。G1 采用`增量回收`的方式，每次回收一些区块，而不是整堆回收。要清楚 G1 不是一个实时收集器（只是接近实时），它会尽力满足我们的停顿时间要求，但也不是绝对的，它基于之前垃圾收集的数据统计，估计出在用户指定的停顿时间内能收集多少个区块。

G1与CMS的特征对比如下：

| 特征                     | G1   | CMS  |
| ------------------------ | ---- | ---- |
| 并发和分代               | 是   | 是   |
| 最大化释放堆内存         | 是   | 否   |
| 低延时                   | 是   | 是   |
| 吞吐量                   | 高   | 低   |
| 压实                     | 是   | 否   |
| 可预测性                 | 强   | 弱   |
| 新生代和老年代的物理隔离 | 否   | 是   |



**G1具备如下特点：**

•**并行与并发**：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-the-world停顿的时间，部分其他收集器原来需要停顿Java线程执行的GC操作，G1收集器仍然可以通过**并发**的方式让Java程序继续运行。•分代收集•空间整合：与CMS的标记-清除算法不同，G1从整体来看是基于**标记-整理算法**实现的收集器，从局部（两个Region之间）上来看是基于“**复制**”算法实现的。但无论如何，这两种算法都意味着G1运作期间不会产生内存空间碎片，收集后能提供规整的可用内存。**这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC**。•可预测的停顿：这是G1相对于CMS的一个优势，降低停顿时间是G1和CMS共同的关注点。

在G1之前的其他收集器进行收集的范围都是整个新生代或者老年代，而G1不再是这样。在堆的结构设计时，G1打破了以往将收集范围固定在新生代或老年代的模式，G1收集器将整个Java堆划分为多个大小相等的独立区域（Region）。Region是一块地址连续的内存空间，G1模块的组成如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/wtxb0ibmp7021WJ1ew935QGgM1MsaO867912qDfaR7GWiczB5QFEZibTZeBD1sxUKGWJYFAOQTyPfrA2ia5IiadblgA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合。Region的大小是一致的，数值是在`1M到32M`字节之间的一个2的幂值数，JVM会尽量划分`2048`个左右、同等大小的Region，这一点可以参看如下源码[1]。其实这个数字既可以手动调整，G1也会根据堆大小自动进行调整。

**G1收集器之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个Java堆中进行全区域的垃圾收集**。G1会通过一个合理的计算模型，计算出每个Region的收集成本并量化，这样一来，收集器在给定了“停顿”时间限制的情况下，总是能选择一组恰当的Regions作为收集目标，让其收集开销满足这个限制条件，以此达到实时收集的目的。

对于打算从CMS或者ParallelOld收集器迁移过来的应用，按照官方[2] 的建议，如果发现符合如下特征，可以考虑更换成G1收集器以追求更佳性能：

•实时数据占用了超过半数的堆空间；•对象分配率或“晋升”的速度变化明显；•期望消除耗时较长的GC或停顿（超过0.5——1秒）。

**G1收集的运作过程大致如下：**

•**初始标记（Initial Marking）**：仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象，**这阶段需要`停顿线程`，但耗时很短**。

•**并发标记（Concurrent Marking）**：是从GC Roots开始堆中对象进行可达性分析，找出存活的对象，**这阶段耗时较长**，但可与用户程序并发执行。

•**最终标记（Final Marking）**：是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，**这阶段需要`停顿线程`，但是可并行执行**。

•**筛选回收（Live Data Counting and Evacuation）**：首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划。这个阶段也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。

全局变量和栈中引用的对象是可以列入根集合的，这样在寻找垃圾时，就可以从根集合出发扫描堆空间。在G1中，引入了一种新的能加入根集合的类型，就是`记忆集`（Remembered Set）。Remembered Sets（也叫RSets）用来跟踪对象引用。G1的很多开源都是源自Remembered  Set，例如，它通常约占Heap大小的20%或更高。并且，我们进行对象复制的时候，因为需要扫描和更改Card  Table的信息，这个速度影响了复制的速度，进而影响暂停时间。

![图片](https://mmbiz.qpic.cn/mmbiz_png/wtxb0ibmp7021WJ1ew935QGgM1MsaO867iczu65CbHrpYUOdZDq3d9q1xGIjqKOjT4U8nz9lvNibmxTmLZicTlsvOw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

G1 比 ParallelOld 和 CMS 会需要更多的内存消耗，那是因为有部分内存消耗于簿记（accounting）上，如以下两个数据结构：

•Remembered Sets：每个区块都有一个 RSet，用于记录进入该区块的对象引用（如区块 A 中的对象引用了区块 B，区块 B 的 Rset 需要记录这个信息），它用于实现收集过程的并行化以及使得区块能进行独立收集。•Collection Sets：将要被回收的区块集合。GC 时，在这些区块中的对象会被复制到其他区块中，总体上 Collection Sets 消耗的内存小于 1%。

### 卡表（Card Table）

有个场景，老年代的对象可能引用新生代的对象，那标记存活对象的时候，需要扫描老年代中的所有对象。因为该对象拥有对新生代对象的引用，那么这个引用也会被称为GC Roots。那不是得又做全堆扫描？成本太高了吧。

HotSpot给出的解决方案是一项叫做`卡表`（Card Table）的技术。该技术将整个堆划分为一个个大小为512字节的卡，并且维护一个卡表，用来存储每张卡的一个标识位。这个标识位代表对应的卡是否可能存有指向新生代对象的引用。如果可能存在，那么我们就认为这张卡是脏的。

在进行Minor GC的时候，我们便可以不用扫描整个老年代，而是在卡表中寻找脏卡，并将脏卡中的对象加入到Minor GC的GC Roots里。当完成所有脏卡的扫描之后，Java虚拟机便会将所有脏卡的标识位清零。

想要保证每个可能有指向新生代对象引用的卡都被标记为脏卡，那么Java虚拟机需要截获每个引用型实例变量的写操作，并作出对应的写标识位操作。

**卡表能用于减少老年代的全堆空间扫描，这能很大的提升GC效率**。

## 3 ZGC

ZGC(Z Garbage Collector)作为一种比较新的收集器，目前还没有得到大范围的关注。作为一款低延迟的垃圾收集器，它有如下几个亮点：

•停顿时间不会超过 **10ms**

•停顿时间**不会随着堆的增大而增大**（控制停顿时间在10ms内）

•支持堆的大小范围很广（**8MB-16TB**）

在ZGC中，连逻辑上的也是重新定义了堆空间（不区分年轻代和老年代），只分为一块块的page，每次进行GC时，都会对page进行压缩操作，所以没有碎片问题。虽然ZGC属于很新的GC技术, 但优点不一定真的出众，ZGC只在特定情况下具有绝对的优势, 如`巨大的堆和极低的暂停需求`。而实际上大多数开发在这两方面都不太成问题(尤其是在服务器端), 而对GC的性能/效率更在意。也有一种观点认为ZGC是为大内存、多cpu而生，它通过分区的思路来降低STW。

ZGC在JDK14前只支持Linux, 从JDK14开始支持Mac和Windows。

**JVM内存参数设置简述**  

```
    -Xms: 初始堆大小，JVM 启动的时候，给定堆空间大小。 

    -Xmx: 最大堆大小，JVM 运行过程中，如果初始堆空间不足的时候，最大可以扩展到多少。 

    -Xmn：设置年轻代大小。整个堆大小=年轻代大小+年老代大小+持久代大小。持久代一般固定大小为 64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大， Sun 官方推荐配置为整个堆的 3/8。 

    -Xss：设置每个线程的 Java 栈大小。JDK5.0 以后每个线程 Java 栈大小为 1M，以前每个线程堆栈大小为 256K。根据应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在 3000~5000 左右。 

    -XX:NewSize=n: 设置年轻代大小 

    -XX:NewRatio=n: 设置年轻代和年老代的比值。如: -XX:NewRatio=3，表示年轻代与年老代比值为 1：3，年轻代占整个年轻代+年老代和的 1/4 

    -XX:SurvivorRatio=n: 年轻代中 Eden 区与两个 Survivor 区的比值。注意 Survivor 区有两个。 如：3，表示 Eden：Survivor=3：2，一个 Survivor 区占整个年轻代的 1/5 。

    -XX:MaxPermSize=n:设置持久代大小
    
    -XX:MaxTenuringThreshold：设置垃圾对象最大年龄。如果设置为 0 的话，则年轻代对象不经过 Survivor 区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在 Survivor 区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。
```

 类似日志的配置信息。会有控制台相关信息输出。 **商业项目上线的时候，不允许使用**。 

​       一定使用 loggc   

​       -XX:+PrintGC   

​       -XX:+Printetails   

​       -XX:+PrintGCTimeStamps  

​       -Xloggc:filename   

## 4 总结

查了下度娘有关G1的文章，绝大部分文章对G1的介绍都是停留在JDK7或更早期的实现很多结论已经存在较大偏差了，甚至一些过去的GC选项已经不再推荐使用。举个例子，JDK9中JVM和GC日志进行了重构，如PrintGCDetails已经被标记为废弃，而PrintGCDateStamps已经被移除，指定它会导致JVM无法启动。

本文对CMS和G1的介绍绝大部分内容也是基于JDK7，新版本中的内容有一点介绍，倒没做过多介绍，后面有机会可以再出专门的文章来重点介绍。通过分析CMS、G1的优缺点，就能更清楚ZGC的由来及优势，特别是停顿时间不超过10ms，是不是已经摩拳擦掌，跃跃欲试了

 **一、Tomcat 本身优化**

 Tomcat 的自身参数的优化，这块很像 ApacheHttp Server。修改一下 xml 配置文件中的参数，调整最大连接数，超时等。此外，我们安装 Tomcat 是，优化就已经开始了。

 1、工作方式选择

 为了提升性能，首先就要对代码进行动静分离，让 Tomcat 只负责 jsp 文件的解析工作。如采用 Apache 和 Tomcat  的整合方式，他们之间的连接方案有三种选择，JK、http_proxy 和 ajp_proxy。相对于 JK  的连接方式，后两种在配置上比较简单的，灵活性方面也一点都不逊色。但就稳定性而言不像JK 这样久经考验，所以建议采用 JK 的连接方式。 
 

 2、Connector 连接器的配置

 之前文件介绍过的 Tomcat 连接器的三种方式： bio、nio 和 apr，三种方式性能差别很大，apr 的性能最优， bio  的性能最差。而 Tomcat 7 使用的 Connector 默认就启用的 Apr 协议，但需要系统安装 Apr 库，否则就会使用 bio  方式。

 3、配置文件优化
 

 配置文件优化其实就是对 server.xml 优化，可以提大大提高 Tomcat 的处理请求的能力，下面我们来看 Tomcat 容器内的优化。

 默认配置下，Tomcat 会为每个连接器创建一个绑定的线程池（最大线程数 200），服务启动时，默认创建了 5 个空闲线程随时等待用户请求。

 首先，打开 ${TOMCAT_HOME}/conf/server.xml，搜索【<Executor name="tomcatThreadPool"】，开启并调整为

```java
 <      Executor   name = "tomcatThreadPool"    namePrefix="catalina-exec-"          maxThreads = "500"     minSpareThreads="20"       maxSpareThreads="50"       maxIdleTime      ="60000"/>  
```

​     

 注意， Tomcat 7 在开启线程池前，一定要安装好 Apr 库，并可以启用，否则会有错误报出，shutdown.sh 脚本无法关闭进程。

 然后，修改<Connector …>节点，增加 executor 属性，搜索【port="8080"】，调整为

​     

```java
<      Connector       executor="tomcatThreadPool"          port="8080"       protocol="HTTP/1.1"     URIEncoding="UTF-8"    connectionTimeout="30000"                        enableLookups="false"     disableUploadTimeout="false" connectionUploadTimeout      ="150000"     acceptCount="300"       keepAliveTimeout="120000"                        maxKeepAliveRequests="1"    compression="on"     compressionMinSize="2048"                        compressableMimeType= "text/html,text/xml,text/javascript,text/css,text/plain,image/gif,image/jpg,image/png"       redirectPort="8443"       />   
```

maxThreads :Tomcat 使用线程来处理接收的每个请求，这个值表示 Tomcat 可创建的最大的线程数，默认值是 200

minSpareThreads：最小空闲线程数，Tomcat 启动时的初始化的线程数，表示即使没有人使用也开这么多空线程等待，默认值是 10。

maxSpareThreads：最大备用线程数，一旦创建的线程超过这个值，Tomcat 就会关闭不再需要的 socket 线程。

上边配置的参数，最大线程 500（一般服务器足以），要根据自己的实际情况合理设置，设置越大会耗费内存和 CPU，因为 CPU  疲于线程上下文切换，没有精力提供请求服务了，最小空闲线程数 20，线程最大空闲时间 60  秒，当然允许的最大线程连接数还受制于操作系统的内核参数设置，设置多大要根据自己的需求与环境。当然线程可以配置在“tomcatThreadPool”中，也可以直接配置在“Connector”中，但不可以重复配置。

URIEncoding：指定 Tomcat 容器的 URL 编码格式，语言编码格式这块倒不如其它 WEB 服务器软件配置方便，需要分别指定。

connnectionTimeout： 网络连接超时，单位：毫秒，设置为 0 表示永不超时，这样设置有隐患的。通常可设置为 30000 毫秒，可根据检测实际情况，适当修改。

enableLookups： 是否反查域名，以返回远程主机的主机名，取值为：true 或 false，如果设置为false，则直接返回IP地址，为了提高处理能力，应设置为 false。

disableUploadTimeout：上传时是否使用超时机制。

connectionUploadTimeout：上传超时时间，毕竟文件上传可能需要消耗更多的时间，这个根据你自己的业务需要自己调，以使Servlet有较长的时间来完成它的执行，需要与上一个参数一起配合使用才会生效。

acceptCount：指定当所有可以使用的处理请求的线程数都被使用时，可传入连接请求的最大队列长度，超过这个数的请求将不予处理，默认为100个。

keepAliveTimeout：长连接最大保持时间（毫秒），表示在下次请求过来之前，Tomcat 保持该连接多久，默认是使用 connectionTimeout 时间，-1 为不限制超时。
maxKeepAliveRequests：表示在服务器关闭之前，该连接最大支持的请求数。超过该请求数的连接也将被关闭，1表示禁用，-1表示不限制个数，默认100个，一般设置在100~200之间。

compression：是否对响应的数据进行 GZIP 压缩，off：表示禁止压缩；on：表示允许压缩（文本将被压缩）、force：表示所有情况下都进行压缩，默认值为off，压缩数据后可以有效的减少页面的大小，一般可以减小1/3左右，节省带宽。

compressionMinSize：表示压缩响应的最小值，只有当响应报文大小大于这个值的时候才会对报文进行压缩，如果开启了压缩功能，默认值就是2048。
compressableMimeType：压缩类型，指定对哪些类型的文件进行数据压缩。

noCompressionUserAgents="gozilla, traviata"： 对于以下的浏览器，不启用压缩。

 如果已经对代码进行了动静分离，静态页面和图片等数据就不需要 Tomcat 处理了，那么也就不需要配置在 Tomcat 中配置压缩了。
 

 以上是一些常用的配置参数属性，当然还有好多其它的参数设置，还可以继续深入的优化，HTTP Connector 与 AJP Connector 的参数属性值，可以参考官方文档的详细说明：

 https://tomcat.apache.org/tomcat-7.0-doc/config/http.html

 https://tomcat.apache.org/tomcat-7.0-doc/config/ajp.html

