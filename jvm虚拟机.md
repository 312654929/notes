

## 什么是java虚拟机

Java 虚拟机是一个可以执行 Java 字节码的虚拟机进程。Java 源文件被编译成能被 Java 虚拟机执行的字节码文件。

Java 被设计成允许应用程序可以运行在任意的平台，而不需要程序员为每一个平台单独重写或者是重新编译。Java 虚拟机让这个变为可能，因为它知道底层硬件平台的指令长度和其他特性。

**JDK：**java开发工具包，包含了jre

**JRE：**java运行环境

![img](http://picture.tjtulong.top/%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%86%85%E5%AD%98.png)

## java堆

![image-20200220212729637](/Users/changwenchao/Library/Application Support/typora-user-images/image-20200220212729637.png)

## 运行时常量

每一个运行时常量池都在java虚拟机的**方法区**中分配。
例如在Java中字符串的创建会在常量池（方法区中StringTable：HashSet）中进行：

```java
public class Changliang {
    public static void main(String[] args) {
        // s1与s2是相等的，为字节码常亮
        String s1 = "abc";
        String s2 = "abc";
		
        // s3创建在堆内存中
        String s3 = new String("abc");
		
        // intern方法可以将对象变为运行时常量
        // intern是一个native方法
        System.out.println(s1 == s3.intern()); // true
    }
}
```

## 对象的创建

<img src="/Users/changwenchao/Library/Application Support/typora-user-images/image-20200220213708799.png" alt="image-20200220213708799" style="zoom:67%;" />

## 在堆中给对象分配内存

两种方式：指针碰撞和空闲列表。我们具体使用的哪一种，就要看我们虚拟机中使用的是什么垃圾回收机制了，如果有压缩整理，可以使用指针碰撞的分配方式。
**指针碰撞：**假设Java堆中内存是绝对规整的，所有用过的内存度放一边，空闲的内存放另一边，中间放着一个指针作为分界点的指示器，所分配内存就仅仅是把哪个指针向空闲空间那边挪动一段与对象大小相等的举例，这种分配方案就叫指针碰撞
**空闲列表：**有一个列表，其中记录中哪些内存块有用，在分配的时候从列表中找到一块足够大的空间划分给对象实例，然后更新列表中的记录，这就叫做空闲列表。

![image-20200301144729547](/Users/changwenchao/Library/Application Support/typora-user-images/image-20200301144729547.png)

年轻代上的内存分配是这样的，年轻代可以分为3个区域：Eden区（伊甸园，亚当和夏娃偷吃禁果生娃娃的地方，用来表示内存首次分配的区域，再 贴切不过）和两个存活区（Survivor 0 、Survivor 1）

1. 绝大多数刚创建的对象会被分配在Eden区，其中的大多数对象很快就会消亡。Eden区是连续的内存空间，因此在其上分配内存极快；
2. 当Eden区满的时候，执行Minor GC，将消亡的对象清理掉，并将剩余的对象复制到一个存活区Survivor0（此时，Survivor1是空白的，==两个Survivor总有一个是空白的==）；
3. 此后，每次Eden区满了，就执行一次==Minor GC==，并将剩余的对象都添加到Survivor0；
4. 当Survivor0也满的时候，将其中仍然活着的对象直接复制到Survivor1，以后Eden区执行Minor GC后，就将剩余的对象添加Survivor1（此时，Survivor0是空白的）。
5. 当两个存活区切换了几次（HotSpot虚拟机默认15次，用-XX:MaxTenuringThreshold控制，大于该值进入老年代）之后，仍然存活的对象（其实只有一小部分，比如，我们自己定义的对象），将被复制到老年代。

### 老年代gc

*可能存在年老代对象引用新生代对象的情况，如果需要执行Young GC，则可能需要查询整个老年代以确定是否可以清理回收，这显然是低效的。解决的方法是，年老代中维护一个512 byte的块——”card table“，所有老年代对象引用新生代对象的记录都记录在这里。Young GC时，只要查这里即可，不用再去查全部老年代，因此性能大大提高。*

老年代存储的对象比年轻代多得多，而且不乏大对象，对老年代进行内存清理时，如果使用停止-复制算法，则相当低效。一般，老年代用的算法是标记-整理算法，即：标记出仍然存活的对象（存在引用的），将所有存活的对象向一端移动，以保证内存的连续。

在发生Minor GC时，虚拟机会检查每次晋升进入老年代的大小是否大于老年代的剩余空间大小，如果大于，则直接触发一次Full GC，否则，就查看是否设 置了-XX:+HandlePromotionFailure（==允许担保失败==），**如果允许，则只会进行MinorGC，此时可以容忍内存分配失败；如果不 允许，则仍然进行Full GC（这代表着如果设置-XX:+Handle PromotionFailure，则触发MinorGC就会同时触发Full GC，哪怕老年代还有很多内存，所以，最好不要这样做）**。