### 堆内存相关

- 堆内存过大一般使用垃圾回收
- 垃圾回收无效情况下
  - jps 查找占用线程
  - jmap具体诊断
  - jconsole jvisualvm 堆转储具体分析方法或者类

### 方法区相关

- 方法区存放类相关filed,方法，类加载器和常量池
- 1.8以前oracle hotSpot的实现方法区放在堆内存，1.8以后类和类加载器移到了操作系统的本地内存
- 永久代内存溢出由堆空间限制，元空间又系统内存限制

- spring、mybatis的二进制字节码的加载是通过cglib这个包做的，都是继承了很早之前的ClassVisitor，都是在运行期间动态生成二进制字节码，动态得类加载，那就再运行期间一定会大量生成类，是有可能溢出的

- 常量池的作用是一张常量表，执行jvm指令的时候，帮助指令找到执行的类名方法名参数类型字面量

- 运行时常量池就是类被加载以后，常量池信息放入内存，且里面符号地址变成真实地址

- 字符串常量相加在编译期间会做优化，而变量相加则原始api是StringBuider.append()

- intern(),有不会覆盖，没有就放  ，1.8以前会复制一份放进串池

- stringTable1.6在永久代，1.8在堆内存
- 字符串入池可以节省空间，intern()效率提升可以增加桶的数量

### 直接内存相关

- nio（non-blocking io）可以使用直接内存加快速度,其原理是放在jvm可以操作的直接内存，相比IO免去了一次数据从系统内存复制到堆区操作时间
- 直接内存的弊端在于jvm不可直接进行GC，原因是其不受jvm内存管理
- 直接内存的释放主要通过Unsafe类直接释放，再byteBuffer的实现类内部，使用了Cleanner来监测对象，一旦ByteBuffer被垃圾回收，那么某个回调方法就会调用其成员变量Unsafe调用freeMomory释放内存
- 禁用显示垃圾回收的前提下，一般通过反射获取Unsafe然后.freeMomory()

## 垃圾回收

- 垃圾回收方法主要有引用计数法和可达性分析算法，jvm使用的是后者
- 引用计数法在循环引用的时候不能及时释放内存
- root对象不可回收，root对象相关联引用根据一定的原则回收
- 强引用 必须所有强引用断开才会触发GC
- 软引用 必须内存不足的时候才会触发GC，可配合引用队列
- 弱以用 不必内存不足也可触发GC，可配合引用队列
- 虚引用 主要配合ByteBuffer使用，必须入队，然后相关线程会调用方法来释放直接内存
- 终结器引用，入队以后又优先级比较低的线程来触发GC，且触发时候再调用finalize()就是说第二次GC才会释放内存

- 软应用非核心逻辑中的部分内容可以使用软引用，其回收是一次回收失败，第二次直接清除软引用对象，可能造成空指针，可以使用引用队列避免
- 垃圾回收算法有标记清除算法、标记整理法
- 标记清除法有点速度快，缺点是会造成内存碎片

- 标记整理法优势无内存碎片，缺点速度慢
- 复制算法有点 不会有内存碎片，缺点占用双倍内存空间

- 垃圾回收流程
  - 对象首先分配在伊甸园
  - 新生代空间不足的时候，出发minor gc,伊甸园和from存活的对象复制到to区，存活对象年龄+1,并交换from to 
  - minor gc 会触发stop the world 暂停其他用户线程，等垃圾回收结束，用户线程才恢复运行
  - 当对象寿命超过阈值，会晋升到老年代，最大寿命15
  - 当老年代空间不足，会先尝试出发minor gc，如果之后空间仍不足，出发full GC(STW 时间更长)

### 垃圾回收器

- 垃圾回收器有串行、吞吐量优先、响应时间优先等三个策略

- 串行
  - 单线程
    堆内存较小，适合个人电脑
- 吞吐量优先
  - 让单位时间内，STW 的时间最短 0.2 0.2 = 0.4，垃圾回收时间占比最低，这样就称吞吐量高	

- 响应时间优先
  - 多线程
    堆内存较大，多核 cpu
    尽可能让单次 STW 的时间最短 0.1 0.1 0.1 0.1 0.1 = 0.5

- 串行垃圾回收器在新生代采用的是复制算法，老年代采用的是标记整理算法

  ![image-20210303170230421](C:\Users\liuzhiji\AppData\Roaming\Typora\typora-user-images\image-20210303170230421.png)

- 吞吐量优先垃圾回收器采用了和串行垃圾回收器一样的算法，且jdk 1.8默认吞吐量优先，并行处理

![image-20210303171035167](C:\Users\liuzhiji\AppData\Roaming\Typora\typora-user-images\image-20210303171035167.png)

- 响应时间优先垃圾回收器（并发机制），采用标记清理算法，和用户线程并发执行，但是在老年代碎片太多导致空间不足的时候，并发失败，会以串行方式做一个标记整理算法，jdk 1.8默认

- G1适用于超大堆内存，整体采用标记整理算法，不同区域之间采用的是复制算法，jdk 1.9默认

- G1流程

  - 新生代垃圾回收
    - stw
    - 复制算法到幸存区
    - 幸存区晋升到老年代，不够年龄就拷贝到from

  - 新生代+并发
    - 初始标记：标记根对象
    - 并发标记：老年代占用堆空间比例达到一定比例阈值，进行并发标记（不会STW）
  - Mixed Collection
    会对 E、S、O 进行全面垃圾回收
    最终标记（Remark）会 STW
    拷贝存活（Evacuation）（这里对老年代会回收最多垃圾得区，并不会拷贝所有得老年代）会 STW

- Full GC
  SerialGC
  新生代内存不足发生的垃圾收集 - minor gc
  老年代内存不足发生的垃圾收集 - full gc
  ParallelGC
  新生代内存不足发生的垃圾收集 - minor gc
  老年代内存不足发生的垃圾收集 - full gc
  CMS
  新生代内存不足发生的垃圾收集 - minor gc
  老年代内存不足
  G1
  新生代内存不足发生的垃圾收集 - minor gc
  老年代内存不足

  再老生代内存超过45%进行并发标记但是打扫速度更快不会触发FullGC，反之会

- 新生代minor gc时候得卡表技术

  在寻找GC Root得时候需要先找到老生代区得一些脏卡 提前记录这些脏卡就可以每次只遍历脏卡

- G1 remark阶段要点
  - 写屏障
  - 放入队列等待重新标记

- G1 优化
  - JDK 8u20 字符串去重
    优点：节省大量内存
    缺点：略微多占用了 cpu 时间，新生代回收时间略微增加
    -XX:+UseStringDeduplication
    将所有新分配的字符串放入一个队列
    当新生代回收时，G1并发检查是否有字符串重复
    如果它们值一样，让它们引用同一个 char[]
    注意，与 String.intern() 不一样
    String.intern() 关注的是字符串对象
    而字符串去重关注的是 char[]
    在 JVM 内部，使用了不同的字符串表
  - JDK 8u40 并发标记类卸载
    所有对象都经过并发标记后，就能知道哪些类不再被使用，当一个类加载器的所有类都不再使用，则卸
    载它所加载的所有类 -XX:+ClassUnloadingWithConcurrentMark 默认启用
    String s1 = new String("hello"); // char[]{'h','e','l','l','o'}
    String s2 = new String("hello"); // char[]{'h','e','l','l','o'}
  - JDK 8u60 回收巨型对象
    一个对象大于 region 的一半时，称之为巨型对象
    G1 不会对巨型对象进行拷贝
    回收时被优先考虑
    G1 会跟踪老年代所有 incoming 引用，这样老年代 incoming 引用为0 的巨型对象就可以在新生
    代垃圾回收时处理掉
  - 动态调整老生代占内存比例触发并发标记的阈值，避免过于频繁得fullGC

## GC调优

- 最好得GC自然是不发生GC，如果因为STW老是卡，第一就应该扪心自问数据是不是太多了，就是核心逻辑得相关类，相关得数据结构是不是太臃肿了
  - 数据太多 比如数据库查询查询了太多冗余数据，要限制查询条数之类
  - 对象图，用到哪个查哪个
  - 对象大小是不是太大了，Integer int能用基本类型不用包装型
  - 内存泄漏问题，内存溢出问题，比如一个static Map一直放，比较好的是不重要的逻辑数据设置成软应用或者弱引用，或者使用第三方缓存技术，redis之类

- 新生代调优
  - 新生代的特点
    所有的 new 操作的内存分配非常廉价
    TLAB thread-local allocation buffer
    死亡对象的回收代价是零
    大部分对象用过即死
    Minor GC 的时间远远低于 Full GC 
  - 新生代在某个阈值（25%到50%）越大越好
    - 压缩老年代，在一个阈值之后，之后的GC就有很多full gc了
    - 吞吐量不会随着新生代的变大一直变大，在某个点其实就会下降，原因是新生代空间一大，复制算法的复制阶段时间耗费也会变多，标记时间带来的红利就没那么大优势了
    - 新生代能容纳所有【并发量 * (请求-响应)】的数据
  - 幸存区达到能保留 活跃对象和需要晋升对象比较合适
  - 晋升阈值配置得当，让长时间存活对象尽快晋升

- 老年代调优
  - CMS老年代的内存越大越好，因为要避免浮动垃圾导致的并发回收失败而退化成串行的垃圾回收器	
  - 先尝试不做调优，如果没有 Full GC 那么已经...，否则先尝试调优新生代
    观察发生 Full GC 时老年代内存占用，将老年代内存预设调大 1/4 ~ 1/3
    -XX:CMSInitiatingOccupancyFraction=percent


- minor GC频繁先思考扩大新生代空间比例

- CMS单次full GC高峰时期时间特别长考虑重新标记以前清理垃圾
  - 一般问题出在重新标记，需要在CMS 一个ScavengeBeforeRemark清理一下垃圾

- 老年代充裕的时候，发生full GC(cms jdk.1.7)
  - 永久代空间不足，增加永久代空间