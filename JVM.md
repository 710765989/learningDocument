# JVM

## JAVA内存区域

- 类装载子系统
  - 通过类装载子系统将编译好的class文件装载到运行时数据区中
- 字节码执行引擎
  - 通过字节码执行引擎来执行内存中的代码

### JDK1.8之前

![](https://github.com/710765989/learningDocument/blob/main/img/java-runtime-data-areas-jdk1.7.png)

### JDK1.8之后

![](https://github.com/710765989/learningDocument/blob/main/img/java-runtime-data-areas-jdk1.8.png)

### 程序计数器

程序计数器（PC寄存器）是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成。

为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储

**主要作用**

- 字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
- 在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。

**生命周期**：随着线程的创建而创建，随着线程的死亡而死亡。

程序计数器是唯一一个不会出现 `OutOfMemoryError` 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。



### Java 虚拟机栈

**生命周期**：随着线程的创建而创建，随着线程的死亡而死亡。

栈由多个栈帧组成，每个栈帧中都拥有：局部变量表、操作数栈、动态链接、方法返回地址。和数据结构上的栈类似，两者都是先进后出的数据结构，只支持出栈和入栈两种操作。

![](https://github.com/710765989/learningDocument/blob/main/img/stack-area.png)

#### 局部变量表

主要存放了编译期可知的各种数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

- 基本数据类型：直接存放值
- 对象类型：存放引用地址

#### 操作数栈

主要作为方法调用的中转站使用，用于**存放方法执行过程中产生的中间计算结果**。另外，**计算过程中产生的临时变量**也会放在操作数栈中。

存放内容

- 中间计算结果

- 临时变量

#### 动态链接

主要服务一个方法需要调用其他方法的场景。在 Java 源文件被编译成字节码文件时，所有的变量和方法引用都作为符号引用（Symbilic Reference）保存在 Class 文件的常量池里。当一个方法要调用其他方法，需要将常量池中指向方法的符号引用转化为其在内存地址中的直接引用。动态链接的作用就是为了将符号引用转换为调用方法的直接引用。

作用

- 将符号引用转换为调用方法的直接引用

简单来说就是：将调用的方法名转换成它所在的地址



栈空间虽然不是无限的，但一般正常调用的情况下是不会出现问题的。不过，如果函数调用陷入无限循环的话，就会导致栈中被压入太多栈帧而占用太多空间，导致栈空间过深。那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出 `StackOverFlowError` 错误。

Java 方法有两种返回方式，一种是 return 语句正常返回，一种是抛出异常。不管哪种返回方式，都会导致栈帧被弹出。也就是说， **栈帧随着方法调用而创建，随着方法结束而销毁。无论方法正常完成还是异常完成都算作方法结束。**

除了 `StackOverFlowError` 错误之外，栈还可能会出现`OutOfMemoryError`错误，这是因为如果栈的内存大小可以动态扩展， 如果虚拟机在动态扩展栈时无法申请到足够的内存空间，则抛出`OutOfMemoryError`异常。



程序运行中栈可能会出现两种错误：

- **`StackOverFlowError`：** 若栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出 `StackOverFlowError` 错误。
- **`OutOfMemoryError`：** 如果栈的内存大小可以动态扩展， 如果虚拟机在动态扩展栈时无法申请到足够的内存空间，则抛出`OutOfMemoryError`异常。

#### 方法返回地址

**作用**：存放调用该方法的**程序计数器**的值

一个方法结束，有两种方式

- 正常执行完成
  - 方法正常退出时，**调用者的pc计数器的值作为返回地址，即调用该方法的指令的下一条指令的地址**。

- 非正常退出，抛出异常
  - 而通过异常退出的，返回地址是要通过异常表来确定，栈帧中一般不会保存这部分信息。

无论通过哪种方式退出，在方法退出后都返回到该方法被调用的位置。

本质上，**方法的退出就是当前栈帧出栈的过程**。此时，需要恢复上层方法的局部变量表、操作数栈、将返回值压入调用者栈帧的操作数栈、设置PC寄存器值等，让调用者方法继续执行下去。



### 本地方法栈

和虚拟机栈所发挥的作用非常相似，区别是： **虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务**

在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。



### 堆

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

Java 世界中“几乎”所有的对象都在堆中分配，但是，随着 JIT 编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了。从 JDK 1.7 开始已经默认开启逃逸分析，如果某些方法中的对象引用没有被返回或者未被外面使用（也就是未逃逸出去），那么对象可以直接在栈上分配内存。

**逃逸分析**：

​	对象逃逸分析就是分析对象的动态作用域，当一个对象在一个方法中被定义之后，它很有可能被外部方法所引用，例如作为调用参数传递到其他地方中。

```java
// 对象逃逸
public User test1{			
    User user=new user();			
    user.setId(1)			
    user.setName("张三")			
    return user;			
}			
// 对象未逃逸			
public void test2{			
   User user=new user();			
    user.setId(1);			
    user.setName("张三");			
}			
```

​	JVM对于这种情况可以通过开启逃逸分析参数(-XX:+DoEscapeAnalysis)来优化对象内存分配位置，使其通过标量替换优先分配在栈上(栈上分配)，JDK7之后默认开启逃逸分析，如果要关闭使用参数(-XX:-DoEscapeAnalysis)

**标量替换：**

​	JVM中的栈帧空间是有很多碎片的，没有一整块空间来存放我们的对象。**当一个对象被确定为不逃逸对象的时候，通过逃逸分析确定该对象不会被外部访问，并且对象可以被进一步分解时，JVM不会创建该对象，而是将该对象成员变量分解若干个被这个方法使用的成员变量所代替，这些代替的成员变量在栈帧或寄存器上分配空间**，这样就不会因为没有一大块连续空间导致对象内存不够分配。很大程度的可以让栈内放更多的对象，**前提是栈空间足够，如果不够还是放到堆内的**



Java 堆是**垃圾收集器管理的主要区域**，因此也被称作 **GC 堆（Garbage Collected Heap）**。从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：新生代和老年代；再细致一点有：Eden、Survivor、Old 等空间。进一步划分的目的是更好地回收内存，或者更快地分配内存。

在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：

1. 新生代内存(Young Generation)
2. 老生代(Old Generation)
3. 永久代(Permanent Generation)

下图所示的 Eden 区、两个 Survivor 区 S0 和 S1 都属于新生代，中间一层属于老年代，最下面一层属于永久代。

![](https://github.com/710765989/learningDocument/blob/main/img/hotspot-heap-structure.41533631.png)

**JDK 8 版本之后 PermGen(永久) 已被 Metaspace(元空间) 取代，元空间使用的是直接内存** 

大部分情况，对象都会首先在 Eden 区域分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 S0 或者 S1，并且对象的年龄还会加 1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。

**动态年龄计算**

>Hotspot 遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了 survivor 区的一半时，取这个年龄和 MaxTenuringThreshold 中更小的一个值，作为新的晋升年龄阈值



### 方法区

当虚拟机要使用一个类时，它需要读取并解析 Class 文件获取相关信息，再将信息存入到方法区。方法区会存储已被虚拟机加载的 **类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。

**作用**：存储已被虚拟机加载的 **类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。

方法区和永久代以及元空间的关系就像接口和实现类的关系，方法区为概念/规范，永久代和元空间为其的不同实现方式。

![](https://github.com/710765989/learningDocument/blob/main/img/method-area-implementation.png)

**为什么要将永久代 (PermGen) 替换为元空间 (MetaSpace) 呢?**

1. 整个永久代有一个 JVM 本身设置的固定大小上限，无法进行调整，而元空间使用的是直接内存，受本机可用内存的限制，虽然元空间仍旧可能溢出，但是比原来出现的几率会更小。

> 当元空间溢出时会得到如下错误： `java.lang.OutOfMemoryError: MetaSpace`

你可以使用 `-XX：MaxMetaspaceSize` 标志设置最大元空间大小，默认值为 unlimited，这意味着它只受系统内存的限制。`-XX：MetaspaceSize` 调整标志定义元空间的初始大小如果未指定此标志，则 Metaspace 将根据运行时的应用程序需求动态地重新调整大小。

2. 元空间里面存放的是类的元数据，这样加载多少类的元数据就不由 `MaxPermSize` 控制了, 而由系统的实际可用空间来控制，这样能加载的类就更多了。

3. 在 JDK8，合并 HotSpot 和 JRockit 的代码时, JRockit 从来没有一个叫永久代的东西, 合并之后就没有必要额外的设置这么一个永久代的地方了。

**方法区常用参数有哪些？**

JDK 1.8 之前永久代还没被彻底移除的时候通常通过下面这些参数来调节方法区大小。

```java
-XX:PermSize=N //方法区 (永久代) 初始大小
-XX:MaxPermSize=N //方法区 (永久代) 最大大小,超过这个值将会抛出 OutOfMemoryError 异常:java.lang.OutOfMemoryError: PermGen
```

JDK 1.8 的时候，方法区（HotSpot 的永久代）被彻底移除了（JDK1.7 就已经开始了），取而代之是元空间，元空间使用的是直接内存。下面是一些常用参数：

```java
-XX:MetaspaceSize=N //设置 Metaspace 的初始（和最小大小）
-XX:MaxMetaspaceSize=N //设置 Metaspace 的最大大小
```

与永久代很大的不同就是，如果不指定大小的话，随着更多类的创建，虚拟机会耗尽所有可用的系统内存。



### 运行时常量池

**字符串常量池** 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

HotSpot 虚拟机中字符串常量池的实现是 `src/hotspot/share/classfile/stringTable.cpp` ,`StringTable` 本质上就是一个`HashSet<String>` ,容量为 `StringTableSize`（可以通过 `-XX:StringTableSize` 参数来设置）。

**`StringTable` 中保存的是字符串对象的引用，字符串对象的引用指向堆中的字符串对象。**

JDK1.7 之前，字符串常量池存放在永久代。JDK1.7 字符串常量池和静态变量从永久代移动了 Java 堆中。

**JDK 1.7 为什么要将字符串常量池移动到堆中？**

主要是因为永久代（方法区实现）的 GC 回收效率太低，只有在整堆收集 (Full GC)的时候才会被执行 GC。Java 程序中通常会有大量的被创建的字符串等待回收，将字符串常量池放到堆中，能够更高效及时地回收字符串内存。

### 字符串常量池

**字符串常量池** 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

HotSpot 虚拟机中字符串常量池的实现是 `src/hotspot/share/classfile/stringTable.cpp` ,`StringTable` 本质上就是一个`HashSet<String>` ,容量为 `StringTableSize`（可以通过 `-XX:StringTableSize` 参数来设置）。

**`StringTable` 中保存的是字符串对象的引用，字符串对象的引用指向堆中的字符串对象。**



JDK1.7 之前，字符串常量池存放在永久代。JDK1.7 字符串常量池和静态变量从永久代移动了 Java 堆中。

![](https://github.com/710765989/learningDocument/blob/main/img/method-area-jdk1.6.png)

![](https://github.com/710765989/learningDocument/blob/main/img/method-area-jdk1.7.png)

![](https://github.com/710765989/learningDocument/blob/main/img/method-area-jdk1.8.png)

#### JDK 1.7 为什么要将字符串常量池移动到堆中？

主要是因为永久代（方法区实现）的 GC 回收效率太低，只有在整堆收集 (Full GC)的时候才会被执行 GC。Java 程序中通常会有大量的被创建的字符串等待回收，将字符串常量池放到堆中，能够更高效及时地回收字符串内存。



### 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 错误出现。



## 类加载过程

类的完整生命周期：

![](https://github.com/710765989/learningDocument/blob/main/img/类加载过程-完善.png)

系统加载 Class 类型的文件主要三步：**加载->连接->初始化**。连接过程又可分为三步：**验证->准备->解析**。

![](https://github.com/710765989/learningDocument/blob/main/img/类加载过程.png)

1. **加载**

   1. 通过全类名获取定义此类的二进制字节流
   2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
   3. 在内存中生成一个代表该类的 `Class` 对象，作为方法区这些数据的访问入口

   虚拟机规范上面这 3 点并不具体，因此是非常灵活的。比如：比较常见的就是从 `ZIP` 包中读取（日后出现的 `JAR`、`EAR`、`WAR` 格式的基础）、其他文件生成（典型应用就是 `JSP`）等等。

   **总结：将java文件编译为class文件，通过类加载器加载到`JVM`中运行**

2. **验证**

   ![](https://github.com/710765989/learningDocument/blob/main/img/验证阶段.png)

3. **准备**

   **准备阶段是正式为类变量分配内存并设置类变量初始值的阶段**，这些内存都将在方法区中分配。

   在JDK1.7之前，分配在**永久代（方法区）**中，JDK1.7之后**静态变量**和**字符串常量池**被移动到了**堆**中

   通常情况下，准备阶段是初始化为零值，在加了`final`关键字等情况时，准备阶段会直接赋值

   ![](https://github.com/710765989/learningDocument/blob/main/img/基本数据类型的零值.png)

4. **解析**

   解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

5. **初始化**

   执行初始化方法`<cinit>`

   对于`<clinit> ()` 方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为 `<clinit> ()` 方法是带锁线程安全，所以在多线程环境下进行类初始化的话可能会引起多个线程阻塞，并且这种阻塞很难被发现。



## 对象创建过程

1. **类加载检查**

   检查类是否已经被加载、解析、初始化过。如果没有，则先执行**类加载过程**

2. **分配内存**

   在类加载完成后便确定了对象所需内存大小，有两种分配方式

   **选择哪种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定**。

   - **指针碰撞**
     - 使用场景：堆内存完整
     - 原理：已使用的内存堆放到一边，未使用的在另一边，中间有一个分界指针，分配内存就是将指针移动所需大小
     - 使用该分配方式的GC回收器：Serial，ParNew
   - **空闲列表**
     - 使用场景：碎片内存
     - 原理：JVM会维护一个列表，该列表记录每块可用空间信息，分配时在列表中选取一块足够大的内存空间
     - 使用该分配方式的GC回收器：CMS

   java 堆内存是否规整，取决于 GC 收集器的算法是"标记-清除"，还是"标记-整理"（也称作"标记-压缩"），值得注意的是，复制算法内存也是规整的。

   **内存分配并发问题（补充内容，需要掌握）**

   虚拟机采用两种方式来保证线程安全：

   - **CAS+失败重试**：**虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。**
   - **TLAB**：为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS 进行内存分配

3. **初始化零值**

   将对象初始化为零值（基本类型为0, false等，引用类型为null），保证了对象实例字段在java代码中不赋值就能直接使用

4. **设置对象头**

   初始化零值完成之后，**虚拟机要对对象进行必要的设置**，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。 **这些信息存放在对象头中。** 另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

5. **执行init方法**

   执行对象的`<init>`初始化方法
   
   

## 对象内存布局

在 Hotspot 虚拟机中，对象在内存中的布局可以分为 3 块区域：**对象头**、**实例数据**和**对齐填充**。

**对象头**：**Hotspot 虚拟机的对象头包括两部分信息**，**第一部分用于存储对象自身的运行时数据**（哈希码、GC 分代年龄、锁状态标志等等），**另一部分是类型指针**，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

**实例数据**：**实例数据部分是对象真正存储的有效信息**，也是在程序中所定义的各种类型的字段内容。

**对其填充**：**对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。**因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说就是对象的大小必须是 8 字节的整数倍。而对象头部分正好是 8 字节的倍数（1 倍或 2 倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。



## 对象的访问定位

建立对象就是为了使用对象，我们的 Java 程序通过栈上的 reference 数据来操作堆上的具体对象。对象的访问方式由虚拟机实现而定，目前主流的访问方式有：**使用句柄**、**直接指针**。

### 句柄

如果使用句柄的话，那么 Java 堆中将会划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与对象类型数据各自的具体地址信息。

![](https://github.com/710765989/learningDocument/blob/main/img/access-location-of-object-handle.png)

### 直接指针

如果使用直接指针访问，reference 中存储的直接就是对象的地址。

![](https://github.com/710765989/learningDocument/blob/main/img/access-location-of-object-handle-direct-pointer.png)

## 类加载器

JVM中内置了三个类加载器，除了 `BootstrapClassLoader` 其他类加载器均由 Java 实现且全部继承自`java.lang.ClassLoader`

1. `BootstrapClassLoader`：最顶层的类加载器，由C++实现，负责加载 `%JAVA_HOME%/lib`目录下的 jar 包和类或者被 `-Xbootclasspath`参数指定的路径中的所有类。
2. `ExtensionClassLoader`：主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类，或被 `java.ext.dirs` 系统变量所指定的路径下的 jar 包。
3. `AppClassLoader`：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。
4. `UserClassLoader`：用户自定义类加载器

### 双亲委派模型

类加载时会默认使用**双亲委派模型**，在加载的时候会优先把该请求委派给父类加载器的`loadClass()`进行处理

![](https://github.com/710765989/learningDocument/blob/main/img/classloader_WPS图片.png)

**双亲委派模型的好处**

双亲委派模型保证了 Java 程序的稳定运行，可以避免类的重复加载（JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类），也保证了 Java 的核心 API 不被篡改。

**如果不想用双亲委派模型怎么办？**

自定义加载器的话，需要继承 `ClassLoader` 。如果我们不想打破双亲委派模型，就重写 `ClassLoader` 类中的 `findClass()` 方法即可，无法被父类加载器加载的类最终会通过这个方法被加载。但是，如果想打破双亲委派模型则需要重写 `loadClass()` 方法



## JVM垃圾回收

### 内存分配和回收原则

#### 对象优先在 Eden 区分配

大多数情况下，对象在新生代中 Eden 区分配。当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC。

当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC。执行Minor GC后如果空间充足，还是在Eden区域进行分配，否则通过 **分配担保机制** 提前放到老年代中去。

#### 大对象直接进入老年代

大对象就是需要大量连续内存空间的对象（比如：字符串、数组）。

大对象直接进入老年代主要是为了避免为大对象分配内存时由于分配担保机制带来的复制而降低效率。

#### 长期存活的对象将进入老年代

虚拟机给每个对象一个对象年龄（Age）计数器。

大部分情况，对象都会首先在 Eden 区域分配。如果对象在 Eden 出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间（s0 或者 s1）中，并将对象年龄设为 1(Eden 区->Survivor 区后对象的初始年龄变为 1)。

对象在 Survivor 中每熬过一次 MinorGC,年龄就增加 1 岁，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。



### 死亡对象判断方法

#### 引用计数法

给对象中添加一个引用计数器：

- 每当有一个地方引用它，计数器就加 1；
- 当引用失效，计数器就减 1；
- 任何时候计数器为 0 的对象就是不可能再被使用的。

**这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很难解决对象之间相互循环引用的问题。**

对象之间的相互引用问题：除了对象 `objA` 和 `objB` 相互引用着对方之外，这两个对象之间再无任何引用。但是他们因为互相引用对方，导致它们的引用计数器都不为 0，于是引用计数算法无法通知 GC 回收器回收他们。

#### 可达性分析算法

通过`GC root`为起点，往下进行搜索，与`GC root`没有引用关系的对象将会被回收

##### 哪些对象可以作为 GC Roots ？

- 虚拟机栈(栈帧中的本地变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象

**对象可以被回收，就代表一定会被回收吗？**

在可达性分析中不可达的对象，并非立即处以死刑，而是先进行死亡标记，如果第二次再被标记就会被回收。

### 四种引用类型

- **强引用（StrongReference）**

  - 当我们创建一个对象赋值给引用一个对象时，它就是一个强引用。
  - 就算空间不足，垃圾回收器也不会回收它，而是直接抛出OOM异常

- **软引用（SoftReference）**

  - 当空间不足时，垃圾回收器会回收掉软引用类型的对象
  - 软引用可用来实现内存敏感的高速缓存。
  - 软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。

- **弱引用（WeakReference）**

  - 弱引用比软引用的生命周期更为短暂。
  - 垃圾回收器线程一旦扫描到弱引用对象就会将其回收，不管空间是否充足。
  - 不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。

  ps：`ThreadLocal`中，`ThreadLocalMap`的`Entry`对象就为一个弱引用对象，它继承了`WeakReference<ThreadLocal<?>>`

- **虚引用（PhantomReference）**

  - 虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。
  - **虚引用主要用来跟踪对象被垃圾回收的活动**。
  - 虚引用必须和引用队列（ReferenceQueue）联合使用。
  - 在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**。



***

### 垃圾收集算法

#### 标记-清除

- 
- 回收掉没有标记的对象

缺点：

	1. 效率问题
	1. 空间问题，会产生大量的不连续碎片

#### 标记-复制

- 将内存分为等大两块，每次使用一块
- 使用完成后，将还存活的对象复制到另一块，把当前这块内存直接清理

优点：

​	能使用连续的内存空间

缺点：

​	比较占用内存

#### 标记-整理

- 标记所有不需要回收的对象
- 将存活对象移动到一端
- 清理存活对象边界以外的对象

#### 分代收集

根据新生代和老年代各自的特点选择合适的算法

新生代：**每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。**

老年代：**老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**

***

### 垃圾回收器

![EDFF38A67FA6418F3DCD32C03E8EAF31](https://github.com/710765989/learningDocument/blob/main/img/EDFF38A67FA6418F3DCD32C03E8EAF31.png)

图中展示了7种不同的分代收集器，如果两个收集器之间存在连线，则说明它们可以搭配使用

**新生代收集器**（全都是**复制**算法）：**Serial**、**ParNew**、**Parallel Scavenge**

**老年代收集器**：**CMS（标记-清除）**、**Serial Old（标记-整理）**、**Parallel Old（标记-整理）**

**整堆收集器**：**G1（一个Region中是标记-清除算法，两个Region之间是标记-复制算法）**



#### Serial

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器。它是一个单线程收集器。

它的 **“单线程”** 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ **"Stop The World"** ），直到它收集结束。

**新生代采用标记-复制算法，老年代采用标记-整理算法。**

![](https://github.com/710765989/learningDocument/blob/main/img/46873026.3a9311ec.png)

它**简单而高效（与其他收集器的单线程相比）**。Serial 收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。Serial 收集器对于运行在 Client 模式下的虚拟机来说是个不错的选择。



#### ParNew

**Serial 收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和 Serial 收集器完全一样。**

**新生代采用标记-复制算法，老年代采用标记-整理算法。**

![](https://github.com/710765989/learningDocument/blob/main/img/22018368.df835851.png)

它是许多运行在 Server 模式下的虚拟机的首要选择，除了 Serial 收集器外，只有它能与 CMS 收集器（真正意义上的并发收集器，后面会介绍到）配合工作。

**并行和并发概念补充：**

- **并行（Parallel）** ：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- **并发（Concurrent）**：指用户线程与垃圾收集线程同时执行（但不一定是并行，可能会交替执行），用户程序在继续运行，而垃圾收集器运行在另一个 CPU 上。



#### Parallel Scavenge

**Parallel Scavenge 收集器关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。**

**新生代采用标记-复制算法，老年代采用标记-整理算法。**

![](https://github.com/710765989/learningDocument/blob/main/img/22018368.df835851.png)

**这是 JDK1.8 默认收集器**



#### Serial Old

**Serial 收集器的老年代版本**，它同样是一个单线程收集器。

它主要有两大用途：一种用途是在 JDK1.5 以及以前的版本中与 Parallel Scavenge 收集器搭配使用，另一种用途是作为 CMS 收集器的后备方案。



#### Parallel Old

**Parallel Scavenge 收集器的老年代版本**。

使用多线程和“标记-整理”算法。在注重吞吐量以及 CPU 资源的场合，都可以优先考虑 Parallel Scavenge 收集器和 Parallel Old 收集器。



#### CMS

**CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。**

**CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。**

CMS使用的是三色标记+Incremental Update算法

从名字中的**Mark Sweep**这两个词可以看出，CMS 收集器是一种 **“标记-清除”算法**实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短
- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

![](https://github.com/710765989/learningDocument/blob/main/img/CMS收集器.8a4d0487.png)

从它的名字就可以看出它是一款优秀的垃圾收集器，主要优点：**并发收集、低停顿**。但是它有下面三个明显的缺点：

- **对 CPU 资源敏感；**
- **无法处理浮动垃圾；**
- **它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。**



#### G1

**G1 (Garbage-First) 是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征.**

**在 JDK1.9 版本后成为 `JVM` 的默认垃圾回收算法**

G1使用的是三色标记+snapshot at the begining （SATB）算法

![](https://github.com/710765989/learningDocument/blob/main/img/format,png)

被视为 JDK1.7 中 HotSpot 虚拟机的一个重要进化特征。它具备以下特点：

- **并行与并发**：G1 能充分利用 CPU、多核环境下的硬件优势，使用多个 CPU（CPU 或者 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程执行的 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
- **分代收集**：虽然 G1 可以不需要其他收集器配合就能独立管理整个 GC 堆，但是还是保留了分代的概念。
- **空间整合**：与 CMS 的“标记-清理”算法不同，G1 从整体来看是基于“标记-整理”算法实现的收集器；从局部上来看是基于“标记-复制”算法实现的。
- **可预测的停顿**：这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为 M 毫秒的时间片段内。

G1 收集器的运作大致分为以下几个步骤：

- **初始标记**
- **并发标记**
- **最终标记**
- **筛选回收**

**G1 收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来)** 。这种使用 Region 划分内存空间以及有优先级的区域回收方式，保证了 G1 收集器在有限时间内可以尽可能高的收集效率（把内存化整为零）。

#### ZGC 收集器

与 CMS 中的 ParNew 和 G1 类似，ZGC 也采用标记-复制算法，不过 ZGC 对该算法做了重大改进。

在 ZGC 中出现 Stop The World 的情况会更少



#### 三色标记算法

它是描述追踪式回收器的一种有用的方法，利用它可以推演回收器的正确性。 首先，我们将对象分成三种类型的。

黑色：根对象，或者该对象与它的子对象都被扫描
灰色：对象本身被扫描,但还没扫描完该对象中的子对象
白色：未被扫描对象，扫描完成所有对象之后，最终为白色的为不可达对象，即垃圾对象

当GC开始扫描对象时，按照如下图步骤进行对象的扫描：
根对象被置为黑色，子对象被置为灰色。

![](https://github.com/710765989/learningDocument/blob/main/img/webp)

继续由灰色遍历,将已扫描了子对象的对象置为黑色。

![](https://github.com/710765989/learningDocument/blob/main/img/webp2)

遍历了所有可达的对象后，所有可达的对象都变成了黑色。不可达的对象即为白色，需要被清理。

![](https://github.com/710765989/learningDocument/blob/main/img/webp3)

这看起来很美好，但是如果在标记过程中，应用程序也在运行，那么对象的指针就有可能改变。这样的话，我们就会遇到一个问题：对象丢失问题

我们如何保证应用程序在运行的时候，GC标记的对象不丢失呢？有如下2中可行的方式：

在插入的时候记录对象
在删除的时候记录对象

刚好这对应CMS和G1的2种不同实现方式：
在CMS采用的是增量更新（Incremental update），只要在写屏障（write barrier）里发现要有一个白对象的引用被赋值到一个黑对象 的字段里，那就把这个白对象变成灰色的。即插入的时候记录下来。
