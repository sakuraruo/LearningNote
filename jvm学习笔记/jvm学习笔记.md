一、技术概述

1.Java虚拟机与系统级虚拟机的区别

Java虚拟机只是面向单一应用程序的虚拟机，但是和系统级虚拟机一样，我们也可以为其分配实际的硬件资源，比如最大的内存大小等。Java虚拟机没有使用传统的PC架构，比如现在的HotSpot虚拟机。实际上采用的是 基于栈的指令集架构 ，而我们传统程序设计一般都是基于寄存器的指令集架构。

tips: 通过查看java字节码 会发现局部变量表中存放了this。

2.jvm运行字节码的一些说明

jvm运行字节码时，所有的操作都是围绕两种数据结构，一种是堆栈(本质上时栈结构)，还有一种是队列。如果JVM执行某条指令时，该指令需要对数据进行操作，那么被操作的数据在指令执行前，必须要压到堆栈上，jvm会自动将栈顶数据作为操作数，如果堆栈上的数据需要暂时保存起来，那么它就会被存储到局部变量队列上。

二、什么是即时编译器

jvm会根据当前代码进行判断，当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”。为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器(Just Time Compiler)。

![](C:\Users\ogiso\IdeaProjects\LearningNote\jvm学习笔记\pic\1.jpg)



tips: Jdk1.4时，Sun Classic VM完全退出了历史舞台，取而代之的时至今使用的HotSpot VM，它时目前使用最广泛的虚拟机，拥有热点代码探测技术、准确式内存管理(虚拟机可以知道内存中某个位置的数据具体是什么类型)等技术。

三、虚拟机发展的未来

2018年4月，Oracle Labs公开了最新的GraalVM，它是一种全新的虚拟机，它能够实现所有的语言统一运行在虚拟机中。

四、jvm启动流程探究

JLI_Launch最初的启动类->SelectVersion选择JRE版本->CreateExecutionEnvironment创建JVM执行环境->Load javaVM 加载JVM->JVMinit 初始化JVM->ContinueInNewThread->ContinueInNewThread0->JavaMain真正开始执行Java程序->InitializeJVm初始化虚拟机->loadMainClass加载主类->GetApplicationClass->Post JVMInit->GetStaticMethodID->CallStaticVoidMethod执行主方法->Leave程序执行结束、结束线程、销毁线程。

五、JNI调用本地方法

java有一个JNI机制，它的全程：java native interface，即java本地接口，它允许在java虚拟机内运行的java代码与其他编程语言如(C/C++和汇编语言)编写的程序和库进行交互(在Android开发中比较多)。

六、JVM内存管理

内存区域划分：既然要管理内存，那肯定不是杂乱无章的，jvm对内存的管理采用的是分区治理，不同的内存区域有着各自的职责所在，在虚拟机运行时，内存区域如下：

![image-20220324170033919](C:\Users\ogiso\IdeaProjects\LearningNote\jvm学习笔记\pic\2.png)

程序计数器：

它和我们传统8086 CPU寄存器的工作差不多，因为JVM虚拟机的目的就是实现物理机那样的程序执行；在8086 CPU中，PC作为程序计数器，负责存储内存地址，该地址指向下一条即将执行的指令，每解释完一条指令，PC寄存器就会自动更新为下一条指令的地址，进入下一个指令周期时，就会根据当前地址所指向的指令进行执行。而JVM中的程序计数器可以看作是当前线程所执行字节码的行号指示器，而行号正好就指的是某一条指令，字节码解释器在工作时也会改变这个值，来指定下一条即将执行的指令。

虚拟机栈：

虚拟机栈式一个非常关键的部分，它是一个栈结构，每个方法被执行的时候，java虚拟机都会同步创建一个栈帧(其实就是栈里面的一个元素)，栈帧包括了当前方法的一些信息，比如局部变量表、操作数栈、动态链接、方法出口等。

![image-20220324171525013](C:\Users\ogiso\IdeaProjects\LearningNote\jvm学习笔记\pic\3.png)

局部变量表：就是方法中的局部变量，在class文件中已经定义好了；

操作数栈：就是我们字节码执行时使用到的栈结构；

栈帧：每个栈帧还保存了一个可以执行当前方法所在类的运行时常量池，目的是：当前方法中如果需要调用其他方法时，就能够从运行时常量池中找到对应的符号引用，再将符号引用转换为直接引用，然后就能直接调用对应方法，这就是动态链接；

方法出口：方法该如何结束，是抛出异常还是正常返回；

流程分析：

从字节码文件分析，在编译之后，整个方法的最大操作数栈深度、局部变量表都是确定好的，当我们程序开始执行时，会根据这些信息封装为对应的栈帧。



堆：

堆是整个Java应用程序共享的区域，也是整个虚拟机最大的一块内存空间，此区域的职责就是存放和管理对象和数组。垃圾回收主要也是作用于这一部分内存区域。



方法区：

方法区也是整个Java应用程序共享的区域，它用于存储所有的类信息、常量、静态常量、动态编译缓存等数据。可以大致分为两个部分，一个是类信息表、一个是运行时常量池。

![image-20220324182344520](C:\Users\ogiso\IdeaProjects\LearningNote\jvm学习笔记\pic\4.png)

类信息表存放的时当前应用程序加载的所有类信息，包括类的版本、字段、方法、接口等信息，同时会将编译时生成的常量池数据全部存放到运行时常量池。当然，常量也并不是只能从类信息中获取，在运行时，也有可能会有新的常量加入到常量池中。

！！！jdk7之后字符串常量池从方法区移动到了堆中。



本地方法栈：

等同于方法区，只不过是作用于本地方法。

tips：设置堆内存为-xms -xmx 设置栈容量为-xss



堆外内存：

通过Unsafe类来操作堆外内存(直接内存)



六、垃圾回收机制

对象存活判定算法

引用计数法：

每一个对象都包含一个引用计数器，用于存放引用计数(其实是存放被引用的次数)

每当有一个地方引用此的对象时，引用次数+1

当引用失效(比如离开了局部变量的作用域或者引用被设定为null)，引用计数-1

当引用计数为0时。表示此对象不可能再被使用，因为这时已经没有任何方法可以得到此对象的引用了

引用计数存在互相引用时，哪怕无法被访问，它们的引用计数还是为1，所以引用计数方案不是最好的解决方案

可达性分析算法：

目前比较主流的编程语言(包括Java)，一般都会使用可达性分析算法发来判断对象是否存活，它采用了类似于树结构的搜索机制。

首先每个对象的引用都有机会成为树的根节点(GC ROOT)，可以被选定作为根节点的条件如下：

1）位于虚拟机栈的栈帧中的本地变量表中所引用到的对象(其实就是我们方法中的局部变量)，同样也包括本地方法栈中JNI引用的对象。

2）类的静态成员变量引用的对象。

3）方法区中，常量池里面引用的对象，比如我们之前提到的String对象。

4）被添加了锁的对象(比如synchronized关键字)

5）虚拟机内部需要用到的对象

一旦已经存在的根节点不满足存在的条件时，那么根节点与的对象之间的连接将被断开。此时虽然对象仍存在堆其他对象的引用，但是由于其没有任何根节点引用，所以此对象即可被判定为不再使用。比如某个方法中的局部变量引用，再方法执行完成返回之后。

总结：如果某个对象无法到达任何GC ROOT，则证明该对象不可能再被使用。



最终判定

虽然再经历了可达性分析算法之后基本可以判定哪些对象能够被回收，但是并不代表此对象一定会被回收，我们依然可以在最终判定阶段对其进行挽留。

还记得Object类时提到的finalize方法吗，他就是在第二次确认时执行。

finalize方法并不是在主线程中调用的，而是虚拟机自动建立的一个低优先级到达finalizer线程。finalize方法只会被调用一次。

eg: System.gc 手动申请执行垃圾回收操作。



七、垃圾回收算法

通过对象判定算法，我们可以准确的知道堆中的哪些对象可以被回收了。垃圾收集器会不定期的检查堆中的对象。查看它们是否被回收条件满足。

分代收集机制

如果对堆中的每一个对象都依次进行判断是否需要回收，这样的效率非常差。我们可以对堆中的对象进行分代管理。比如某些对象，在多次垃圾回收时，都未被判定为可回收对象，我们完全可以将这一部分大的对象放到一起，并让垃圾收集器减少回收此区域对象的频率，这样就可以提高垃圾回收的效率了。

Java虚拟机将堆内存划分为新生代、老年代和永久代(其中永久代是HotSpot虚拟机特有的概念，在JDK8之前方法区实际上就采用的是这种永久代作为实现，而在JDK8之后，方法区由元空间实现，并且使用的是本地内存，容量大小取决于物理机实际大小)

新生代和老年代

不同的分代内存回收机制也存在不同之处，在HotSpot虚拟机中，新生代被划分为三块，一块较大的Edan空间和两块较小的survivor空间，默认比例为8：1：1，老年代大的GC频率相对较低，永久代一般存放类信息(其实就是方法区的实现)

![image-20220328145308830](C:\Users\ogiso\IdeaProjects\LearningNote\jvm学习笔记\pic\5.png)



运作过程

首先，所有新创建的对象都会进入到新生代的Edan区(如果是大对象会被直接丢进老年代)，在进行新生代的垃圾回收时，首先会对所有新生代区域的对象进行扫描，并回收那些不再使用的对象。

接着，再一次垃圾回收之后，Edan区域没有被回收的对象，会进入到survivor区。在一开始From和To都是空的，而GC之后，所有的Eden区存活的对象都会直接进入到From区，最后From和To会发生一次交换。这样一直持续到Survivor中的对象年龄大于15，则进入老年代。

垃圾收集划分：

Minor GC 次要垃圾回收，主要进行新生代区域的垃圾回收

触发条件：新生代的Eden区容量已满。

Major GC 主要垃圾回收，主要进行老年代的垃圾收集

Full GC 完全垃圾回收，对整个Java堆内存和方法区进行垃圾回收

触发条件1：每次晋升到老年代的对象平均大小大于老年代剩余空间

触发条件2 Minor GC后存活的对象超过了老年代剩余空间

触发条件3 永久代内存不足(jdk1.8之前就是方法区内存不足)

触发条件4 手动调用System.gc方法

tips：-XX:+PrintGCDetails 添加启动参数查看GC日志



空间分配担保

一般情况下新生代的回收率会很高，基本上不用担心会出现这种情况。在一次GC后，新生代Edan区仍然存在大量对象，但是以及超出Survivor区的容量，这时候就需要用到空间分配担保机制。可以直接把survivor区无法容纳的对象直接送到老年代，让老年代进行分配担保(前提是老年代也能够装下)。如果老年代也装不下了呢，这样首先会判断一下之前的每次垃圾回收进入到老年代的平均大小是否小于大当前老年代的剩余空间，如果小于，那么说明也许可以放得下，否则会先来一次Full GC，进行一次大规模垃圾回收，来尝试腾出空间，再次判断，要是还是装不下则直接抛出OOM错误。



垃圾回收：标记清除、复制、整理算法

标记清除算法

标记出所有需要回收的对象，然后再一次回收掉被标记的对象，或是标记出所有不需要回收的对象，只回收未标记的对象。虽然此方法非常简单，但是缺点也很明显，首先如果在内存中存在大量对象，那么可能就会存在大量的标记，并且大规模进行清除。这就会导致连续的内存空间会出现很多间隙，碎片化会大冬之连续内存空间的利用率降低。

标记复制算法

标记清除算法在面对的大量对象时效率低，那么我们可以采用标记复制算法。它将容量分为同样大小的两块区域。

标记复制算法，实际上就是将内存区域划分为两个大小相同的两块区域，每次只使用其中的一块区域，每次垃圾回收结束后，将所有存活的对象去拿不复制到另一块区域中，并一次性清空当前区域，虽然浪费了一些时间进行复制操作，但是这样能够很好的解决对象大面积回收后空间碎片化严重的问题。

这种算法非常适用于新生代，因为新生代的回收效率极高，一般不会留下太多对象，新生代survivor区其实就是这个思路，8：1：1的比例也是为了对标记复制算法进行优化而采取的。

标记整理算法

虽然标记复制算法能够很好的应对新生代高回收率的场景，但是放到老年代就显得很鸡肋了。一般长期都回收不到大的对象，才有机会进入到老年代，所以老年代都是一些钉子户，可能一次GC后，仍然存在很多对象，而标记复制算法会在GC后完整复制整个区域内容，并且会折损50%的内存，显然这个不适用于老年代。

在标记所有待回收的对象之后，不急着去进行回收操作，而是将所有待回收的对象整齐排列在一段内存空间中，而需要回收的对象全部往后丢，这样，前半部分的所有对象都是无需回收的，后半部分直接一次性清除即可。

虽然这样能保证内存空间充分使用，并且也没有标记复制算法那么复杂，但是缺点也是显而易见的，它的效率比前面两者都低，甚至需要修改对象在内存中的位置，此程序必须要停下来，在这种极端情况下，可能会导致STW。



八、垃圾收集器实现

Serial收集器

这款收集器是元老级别收集器，在jdk1.3.1之前，是虚拟机新生代区域收集器的唯一选择。这是一款单线程的垃圾收集器，也就是说，当开始进行垃圾回收时，需要暂停所有的线程，直到垃圾收集工作结束。它的新生代收集算法采用的是标记复制算法。

虽然缺点很明显，但是优势也是显而易见的。

1.设计简单高效

2.在用户的桌面应用场景中，内存一般不大，可以短时间完成垃圾收集，只要不频繁发生，使用串行回收器时可以接受的。所以在客户端模式(一般一些桌面级图形化界面应用程序)下的新生代中，默认垃圾收集器至今依然时Serial收集器。

ParNew收集器

这款垃圾收集器相当于是Serial收集器的多线程版本，它能够支持多线程垃圾收集。

Parallel Scavenge/Parallel Old收集器

Parallel Scavenge同样是一款面向新生代的垃圾收集器，同样采用标记复制算法实现，在JDK6时也推出了其老年代收集器Parallel Old，采用标记整理算法实现。

和ParNew收集器不同的时，它会自动衡量吞吐量，并根据吞吐量来决定每次垃圾回收的时间，这种自适应机制，能够很好地权衡当前机器的性能，根据性能选择最优方案。

CMS收集器

在jdk1.5，HotSpot推出了一款在强交互应用中几乎可认为有划时代意义的垃圾收集器：CMS(Concurrent-Mark-Sweep)收集器。这款收集器时HotSpot虚拟机中第一款真正意义上的并发(注意这里的并发和之前的并行是有区别的，并发可以理解为同时运行用户线程和GC线程，而并行可以理解为多条GC线程同时工作)，它第一次实现了让垃圾收集线程与用户线程同时工作（但是不代表它不会停止用户线程，在初始阶段进行标记）。

它主要采用标记清除算法：

它的垃圾回收分为四个阶段：

1）初始阶段(需要暂停用户线程)：这个阶段的主要任务仅仅只是标记出GC ROOTS能直接关联到的对象，速度比较快，不用担心会停顿太长时间。

2）并发标记：从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾回收线程一起并发运行。

3）重新标记（需要暂停用户线程）:由于并发标记阶段可能某些用户线程会导致标记产生变更，因此这里需要再次暂停所有线程进行并行标记，这个时间比初始标记时间长一丢丢。

4）并发清除：最后就可以将所有标记好的无用对象进行剔除，因为这些对应用程序也用不到了，所以可以与用户线程并发运行。

虽然它的优点非常大，但是缺点也是显而易见的，标记清除会产生大量的内存碎片，导致可用的连续空间变少，长期这样下来，会有更高的概率触发Full GC，并且在与用户线程并发执行的情况下，也会占用一部分系统资源，导致用户线程的运行速度一定程度上减慢。

不过如果你希望的是最低的GC停顿时间，那么这款垃圾收集器无疑是最佳选择，不过自从G1收集器问世之后，CMS就不再推荐使用了。

Garbage First G1收集器

此垃圾收集器也是一款跨时代的垃圾收集器，在JDK7正式走上历史舞台，它是一款主要面向与服务端的垃圾收集器，并且在jdk9时，取代了jdk8的parallel scavenge+parallel old的回收方案。

垃圾回收分为Minor GC、major GC和Full GC，它们分别代表的是新生代、老年代和整个堆内存的垃圾回收，而G1收集器很巧妙的绕过了这些约定，它将整个java堆划分成2048个相同大小的独立Region块，每个Region块的大小根据堆空间的实际大小定，整体被控制在1-32MB之间，且都为2的N次幂。所有的Region大小相同，且在JVM大的整个生命周期内不会发生改变。

那么分出这些Region有何意义呢？每个Region‘都可以根据需要，自由决定扮演哪个角色(Eden、survivor和老年代，收集器会根据对应的角色不同采用不同的回收策略)。此外，G1收集器还存在一个Humengous区域，专门用于存放大对象(一般认为大小超过了Region一半的对象为大对象)这样，新生代、老年代在物理上，不再是一个连续的内存区域，而是到处分布的。

元空间

Jdk8之前，HotSpot虚拟机的方法区实际上是永久代实现的。在Jdk8之后，HotSpot不再使用永久代，而是采用了全新的元空间。类的元信息被存储在元空间中。元空间没有使用堆内存，而是与堆不相连的本地内存区域。所以，理论上系统可以使用的内存有多大，元空间就有多大，所以不会出现永久代存在时的内存溢出问题。这项改造也是有必要的，永久代的调优很困难，尽管可以设置永久代的大小，但是很难确定一个合适的大小，因为影响因素很多，比如类数量的多少、常量的多少等。

九、其他引用类型

在Java中，如果变量是一个对象类型的，那么它实际上存放的是对象大的引用。但是如果是一个基础类型，那么存放的就是基础类型的值。实际上我们平时代码中类似于Object o =new Object()这样的引用类型，细化 之后可以称为强引用。如果方法中存在这样的强引用类型，现在需要回收强引用所指向的对象，那么要么此方法运行结束，要么引用连接断开，否则被引用的对象是无法判定为可回收的，因为我们说不定后面还要使用它。

当JVM内存空间不足时，JVM宁愿抛出OOM错误使得程序异常终止，也不会靠随意回收具有强引用的“存活”对象来解决内存不足。

除了强引用之外，Java也为我们提供了三种额外的引用类型

软引用

软引用不像强引用那样不可回收，当JVM认为内存不足时，会去试图回收软引用指向的对象，即JVM会确保在抛出OutOfMemoryError之前，清理软引用指向的对象。

可以看到软引用还存在一个带队列的构造方法，软引用可以和一个引用队列(ReferenceQueue)联合使用，如果软引用所引用对象被垃圾回收器回收，java虚拟机就会把这个软引用加入到与之关联的引用队列中。

可以看到，当内存不足时，软引用所指向的对象被回收了。

弱引用

弱引用比软引用的生命周期还要短，在进行垃圾回收时，不管当前内存空间是否充足，都会回收它的内存。

tips:WeakHashMap正是一种类似于弱引用的HashMap类，如果Map中的key没有其他引用那么此Map会自动丢弃此键值对。

虚引用

虚引用相当于没有引用，随时都有可能会被回收。也就是说无论我们调用多少次get方法得到的永远都是null，因为虚引用本身就不算是个引用。相当于这个对象不存在任何引用，并且只能使用带队列构造方法，以便对象被回收时接到通知。

十、类文件结构

魔数：前4个字节，共32位(用于表示这个文件时一个JVM可以运行的字节码文件)

版本号：魔数后四个字节存储的时字节码文件的版本号(Jdk版本)

类常量池:这里面存放了类中所有的常量信息，这里的常量并不是我们手动创建的final类型常量，而是程序与逆行一些需要用到的常量数据，比如字面量和符号引用等。

十一、类加载机制

类加载过程

首先，要加载一个类，一定是出于某种目的，比如我们要运行的Java程序，那么就必须要加载主类才能运行主类中的主方法，又或是我们需要加载数据库驱动，那么可以通过反射来将对应的数据库驱动类进行加载。

一般在这些情况下，如果类没有加载，那么会被自动加载

1）使用new关键字创建对象时

2）使用某个类的静态成员(包括方法与字段时) （final类型的静态字段有可能在编译的时候被放到了当前类的常量池中，这种情况下是不会触发的）

3）使用放射对类信息进行获取的时候(之前数据库驱动就是这样)

4）加载一个类的子类时

5）加载接口的实现类。且接口带有default的方法默认实现时。

类的详细加载流程

加载->（校验->准备->解析）都属于链接 link阶段 ->初始化->使用->卸载





