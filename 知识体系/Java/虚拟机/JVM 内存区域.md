---
title: JVM 内存区域
comments: true
date created: 2023-03-22
date modified: 2023-03-22
id: home
layout: page
tags:
  - JVM
  - 内存区域
  - 内存布局
  - Java
---
# 简介

## 什么是 JVM？

Java 虚拟机（Java Virtual Machine，简称 JVM），它能识别 .class 后缀的字节码文件（Java bytecode），并且能够解析它的指令，最终调用操作系统上的函数以完成指定操作。

## 为什么需要 JVM？

Java 程序使用 javac 编译成 .class 文件之后，还需要使用 Java 命令去主动执行它，操作系统并不认识这些 .class 文件，JVM 则充当了翻译官的角色。JVM 屏蔽了与具体操作系统平台相关的信息，使得 Java 程序只需生成在 Java 虚拟机上运行的目标代码（字节码），就可以在多种平台上不加修改地运行。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120503.png)

从图中可以看到，有了 JVM 这个抽象层之后，Java 就可以实现跨平台了。JVM 只需要保证能够正确执行 .class 文件，就可以运行在诸如 Linux、Windows、MacOS 等平台上了。

JVM 解释的是类似于汇编语言的字节码，需要一个抽象的运行时环境。同时，这个虚拟环境也需要解决字节码加载、自动垃圾回收、并发等一系列问题。JVM 其实是一个规范，定义了 .class 文件的结构、加载机制、数据存储、运行时栈等诸多内容，最常用的 JVM 实现就是 Hotspot。

一个 Java 程序，首先经过 javac 编译成 .class 文件，然后 JVM 将其加载到 `元数据` 区，执行引擎将会通过 `混合模式` 执行这些字节码。执行时，会翻译成操作系统相关的函数。JVM 作为 .class 文件的黑盒存在，输入字节码，调用操作系统函数。
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120537.png)
## JVM、JRE 和 JDK 三者关系

JVM 只是一个翻译官，它负责把字节码翻译成机器码，但是需要注意，JVM 不会自己生成代码，需要大家编写代码，同时需要很多依赖类库，这个时候就需要用到 JRE。

JRE 除了包含 JVM 之外，提供了很多的类库（就是我们说的 jar 包，它可以提供一些即插即用的功能，比如读取或者操作文件，连接网络，使用 I/O 等等之类的）这些东西就是 JRE 提供的基础类库。JVM 标准加上实现的一大堆基础类库，就组成了 Java 的运行时环境，也就是我们常说的 JRE（Java Runtime Environment）。有了 JRE 之后，Java 程序便可以运行来了。

但对于程序员来说，JRE 还不够。还需要编译代码、调试代码、打包代码，有时候还需要反编译代码等等。因此需要使用 JDK，除了 JRE 之外，JDK 还提供了一些非常好用的小工具，比如 javac（编译代码）、java、jar （打包代码）、javap（反编译 < 反汇编 >）等。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120708.png)

# JVM 运行时数据区

相对于 C 和 C++ 的手动内存管理和复杂难以理解的指针等特性，Java 引以为豪的就是它的自动内存管理机制，这使得 Java 程序写起来方便得多。然而这种呼之即来挥之即去的内存申请和释放方式，自然也有它的代价。为了管理这些快速的内存申请释放操作，就必须引入一个池子来延迟这些内存区域的回收操作。另外，Java 程序的数据结构是非常丰富的，比如有静态成员变量、动态成员变量、区域变量、短小紧凑的对象声明和庞大复杂的内存申请等等。这么多不同的数据结构，到底是在什么地方存储的，它们之间又是怎么进行交互的呢？这就有了内存区域划分的概念，如图：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120716.png)

JVM 内存区域划分如图所示，从图中我们可以看出：

- JVM 堆中的数据是共享的，是占用内存最大的一块区域。
- 可以执行字节码的模块叫作执行引擎。
- 执行引擎在线程切换时依靠的就是程序计数器来进行恢复。
- JVM 的内存划分与多线程是息息相关的。程序中运行时用到的栈，以及本地方法栈，它们的维度都是线程。
- 本地内存包含元数据区和一些直接内存。

![](static/boxcnFXQjU3d0urFc9C5qGM6ISd.png)

## 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，由于 JVM 可以并发执行线程，因此会存在线程之间的切换，而这个时候就程序计数器会记录下当前程序执行到的位置，以便在其他线程执行完毕后，恢复现场继续执行。

JVM 会为每个线程分配一个程序计数器，与线程的生命周期相同。

如果线程正在执行的是一个 Java 方法，这个计数器记录的是正在执行虚拟机字节码指令的地址。如果正在执行的是 Native 本地方法，计数器的值则为空。

<strong>程序计数器是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。</strong>

## 虚拟机栈

虚拟机栈描述的是 Java 方法执行的内存模型：

每个方法在执行的同时都会创建一个栈帧（Stack Frame，是方法运行时的基础数据结构）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

虚拟机栈是每个线程独有的，随着线程的创建而存在，线程结束而死亡。

在虚拟机栈内存不够的时候会 OutOfMemoryError，在线程运行中需要更大的虚拟机栈时会出现 StackOverFlowError。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120721.png)

从上图可以看出，虚拟机栈包含很多<strong>栈帧</strong>，每个方法执行的同时会创建一个栈帧，栈帧又存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。

### 虚拟机栈运行的原理

- JVM 直接对 Java 栈的操作只有两个，就是对栈帧的压栈和出栈，遵循"先进后出"的原则。
- 每一次函数调用都会有一个对应的栈帧被压入虚拟机栈中，每一个函数调用结束后，都会有一个栈帧被弹出。
- 在一条活动的线程中，一个时间点上，只会有一个数据的栈帧。即只有当前正在执行的方法的栈帧是有效的。这个栈帧称为当前栈帧（Current Frame），与当前栈帧相对应的方法就是当前方法（Current Method），定义这个方法的类就是当前类（Current Class）。
- 执行引擎运行的所有字节码指令只针对当前的栈帧进行操作。
- 如果该方法调用了其他方法，对应新的栈帧会创建处理，放在栈的顶部，成为新的当前栈。
- 不同线程中所包含的栈帧是不允许相互作用的，即不可能在一个栈帧之中引用另外一个线程的栈帧。
- 如果当前方法调用了其他方法，方法返回之际，当前栈帧会传回此方法的返回结果给前一个栈帧，接着，虚拟机会丢弃当前栈帧，是的前一个栈帧重新成为当前栈帧。
- Java 方法有两种返回方式：return 语句和抛出异常，不管哪种返回方式都会导致栈帧被弹出。

### 局部变量表

局部变量表主要存放了编译期可知的各种数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和 returnAdress 类型（指向了一条字节码指令的地址）。

### 操作数栈

主要用于<strong>保存计算过程的中间结果</strong>，同时作为计算过程中<strong>临时的存储空间</strong>。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作。

操作数栈、局部变量表和程序计数器的工作过程：

![](static/boxcnIP1LPtHC6Df6FWYsQDun3z.png)

### 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用。持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）。

方法调用并不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本(即调用哪一个方法)，这也是 Java 强大的扩展能力，在运行期间才能确定目标方法的<strong>直接引用</strong>。

所有<strong>方法调用中的目标方法</strong>在 Class 文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用。

在 Java 源文件被编译到字节码文件中时，所有的变量和方法引用都作为符号引用（Symbolic Reference）保存在 class 文件的常量池中。比如：描述一个方法调用另外的其他方法时，就是通过常量池中指向方法的引用来表示的，那么动态链接的作用就是为了将这些符号引用转换为调用方法的直接引用。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120904.png)

下面从字节码的角度分析动态连接的过程：

```java
public class DynamicLinkingTest {
    int num = 10;
    public void methodA() {
        System.out.println("methodA...");
    }
    public void method() {
        System.out.println("methodB...");
        methodA();
        num++;
    }
}
```

上面代码反编译以后可以看到如下指令：

```java
 Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokevirtual #2                  // Method a:()V
         4: return
```

invokevirtual 后面的 #2 符号引用对应的就是运行时常量池中的直接引用。

#2 对应了方法引用，#3，#17……最终对应到 methodA：

```java
Constant pool:
   #1 = Methodref          #4.#16         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#17         // com/suanfa/jvm/OperandStackTest.a:()V
   #3 = Class              #18            // com/suanfa/jvm/OperandStackTest
   #4 = Class              #19            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               Lcom/suanfa/jvm/OperandStackTest;
  #12 = Utf8               a
  #13 = Utf8               b
  #14 = Utf8               SourceFile
  #15 = Utf8               OperandStackTest.java
  #16 = NameAndType        #5:#6          // "<init>":()V
  #17 = NameAndType        #12:#6         // a:()V
  #18 = Utf8               com/suanfa/jvm/OperandStackTest
  #19 = Utf8               java/lang/Object
```

#### <strong>方法调用</strong>

在 JVM 中，将符号引用转换为调用方法的直接引用与方法的绑定机制有关。

1. 静态链接、早期绑定：当一个字节码文件别装入 JVM 内部，如果被调用的目标方法在编译器可知，且运行期保持不变时，这种情况下将调用方法的符号引用转换为直接引用的过程称之为静态链接。对应的方法绑定机制为<strong>早期绑定</strong>。
2. 动态链接、晚期绑定：如果方法被调用的时候在编译期无法确定下来，就是说，只能在运行时将调用方法的符号引用转换为直接引用，引用这种引用转换过程具备动态性，因此称之为动态链接。对应的方法绑定机制为<strong>晚期绑定</strong>。

#### <strong>虚方法与非虚方法</strong>

- 如果方法在编译期就确定了具体的调用版本，这个版本在运行时是不可变的。这样的方法称之为非虚方法。
- 静态变量、私有方法、final 方法、实例构造器、父类方法都是非虚方法。
- 其它方法称之为虚方法。

##### 普通调用指令

- invokestatic ：静态方法，解析阶段确定唯一方法版本
- invokespecial ：调用`<init>`方法、私有方法以及父类方法，解析阶段确定唯一方法版本
- invokevirtual ：调用所有虚方法
- invokeinterface ：调用接口方法

##### 动态调用指令

- invokedynamic ： 动态解析所需要调用的方法，然后执行。invokedynamic 指令是在 Java 7 中增加的，为了实现动态类型语言支持而做的一种改进，但在 Java 7 中并没有提供直接生成的 invokedynamic 指令的方法，需要借助 ASM 这种底层字节码工具才能够生成 invokedynamic 指令。直到 JDK 8 的 Lambda 表达式的出现，invokedynamic 指令在 Java 中才有了直接生成的方式。

前四条指令固化在虚拟机的内部，方法的调用执行不可人为干预，而 invokedynamic 指令则支持由用户确定版本。其中 invokevirtual 和 invokestatic 指令调用的方法称为非虚方法，其余的（final 修饰除外）称为虚方法。

下面以具体实例展示以上介绍的各种调用指令：

```java
/**
 * 解析调用中非虚方法、虚方法的测试
 */
class Father {
    public Father(){
        System.out.println("Father 默认构造器");
    }

    public static void showStatic(String s){
        System.out.println("Father show static " + s);
    }

    public final void showFinal(){
        System.out.println("Father show final");
    }

    public void showCommon(){
        System.out.println("Father show common");
    }

}

public class Son extends Father{
    public Son(){
        super();
    }

    public Son(int age){
        this();
    }

    public static void main(String[] args) {
        Son son = new Son();
        son.show();
    }

    //不是重写的父类方法，因为静态方法不能被重写
    public static void showStatic(String s) {
        System.out.println("Son show static " + s);
    }

    private void showPrivate(String s) {
        System.out.println("Son show private " + s);
    }

    public void show() {
        //invokestatic
        showStatic("大头儿子");
        //invokestatic
        super.showStatic("大头儿子");
        //invokespecial
        showPrivate("hello!");
        //invokespecial
        super.showCommon();
        //invokevirtual 因为此方法声明有final，不能被子类重写
        //所以也认为该方法是非虚方法
        showFinal();
        //虚方法如下
        //invokevirtual
        //没有显式加super，被认为是虚方法，因为子类可能重写showCommon
        showCommon();
        info();

        MethodInterface in = null;
        //invokeinterface  
        //不确定接口实现类是哪一个 需要重写
        in.methodA();
        //invokedynamic
        lambda(()-> {
          
        });
    }

    public void info(){

    }
    
    public void lambda(MethodInterface interface) {
        methodInterface.methodA();
    }

}

interface MethodInterface {
    void methodA();
}
```

### 方法出口

指方法返回地址，是存放调用该方法的程序计数器的值。

方法返回分为正常返回和异常退出。无论何种退出情况，都将返回至方法当前被调用的位置，这样程序才能继续执行。

本质上，方法的退出就是当前栈帧出栈的过程。此时需要回复上层方法的局部变量表、操作数栈、将返回值压入调用者的栈帧的操作数栈、设置程序计数器的值等，让调用者方法继续执行下去。正常完成出口和异常完成出口的区别在于：通过异常完成出口退出的不会给它的上层调用者产生任何的返回值。

### 本地方法栈

Java 虚拟机栈是用来为 Java 方法服务，相应地，本地方法栈则是用来为 native 方法服务的，可以认为是通过 JNI (Java Native Interface) 直接调用本地 C/C++ 库，不受 JVM 控制。需要注意的是，HotSpot 虚拟机直接把本地方法栈和虚拟机栈合二为一了。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120910.png)

本地方法栈也会抛出 StackOverflowError 和 OutOfMemoryError 异常。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_120920.png)

## 堆

堆是 JVM 上最大的内存区域，我们申请的<strong>几乎所有的对象</strong>，都是在这里存储的。我们常说的垃圾回收，操作的对象就是堆。

一个对象创建的时候，决定是堆上分配还是在栈上分配，和两个方面有关：对象的类型和在 Java 类中存在的位置。

- 对于普通对象来说，JVM 会首先在堆上创建对象，然后在其他地方使用的其实是它的引用。比如，把这个引用保存在虚拟机栈的局部变量表中。
- 对于基本数据类型来说（byte、short、int、long、float、double、char)，有两种情况。由于每个线程拥有一个虚拟机栈，当在方法体内声明了基本数据类型的对象，就会在栈上直接分配。其他情况，则都是在堆上分配。

堆空间一般是程序启动时，就申请了，但是并不一定会全部使用。

随着对象的频繁创建，堆空间占用的越来越多，就需要不定期的对不再使用的对象进行回收。这个在 Java 中，就叫作 GC（Garbage Collection）。由于对象的大小不一，在长时间运行后，堆空间会被许多细小的碎片占满，造成空间浪费。所以，仅仅销毁对象是不够的，还需要堆空间整理。

现在的虚拟机（包括 HotSpot）都是采用分代回收算法。在分代回收的思想中， 把堆分为：新生代 + 老年代 + 永久代（Java 1.8 之前的方法区的实现方式）； 新生代又分为 Eden + From Survivor + To Survivor 区。几乎所有的 Java 对象都在 Eden 区被 new 出来的。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121108.png)

在 Hotspot 中，Eden、 From 和 To 区空间大小默认比例是 8：1：1。

也可以使用 -XX:SurvivorRatio 调整这个空间比例，比如 -XX:SruvivorRatio=8。

## 方法区

方法区（Method Area）与 Java 堆一样，是所有<strong>线程共享</strong>的内存区域。

方法区用于存储已经被虚拟机加载的类信息（即加载类、接口、枚举、注解时需要的信息，包括版本、field、方法、接口等信息）、final 常量、静态变量、编译器即时编译的代码等。

方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。

### 版本演进

Java 8 之前，许多 Java 程序员都习惯在 HotSpot 虚拟机上开发、部署程序，很多人都更愿意把方法区称呼为“永久代”（Permanent Generation），或者将两者混为一谈。本质上这两者是不等价的，因为仅仅是当时的 HotSpot 虚拟机设计团队把垃圾收集器的分代设计思想扩展到方法区，或者说使用永久代来实现方法区而已，这样使得 Hotspot 的垃圾收集器能够像管理 Java 堆一样管理这部分内存，省去专门为方法区编写内存管理代码的工作。但是对于其他虚拟机（如 JRockit 和 J9）来说是不存在永久代的概念的，因为《Java 虚拟机规范》对方法区的实现要求比较宽松。

在 Java 6 的时候 Hotspot 开发团队就有放弃永久代，逐步改为采用本地内存来实现方法区的计划了。到了 Java 7，Hotspot 已经把原本放在永久代的字符串常量池、静态变量等移动到堆里；而从 Java 8 开始，则放弃了永久代的概念，改用与 JRockit 和 J9 一样在本地内存中实现的元空间（MetaSpace）来代替，并把之前永久代剩余的内容（运行时常量池和其他类信息等）全部移动到元空间中

Java 7 之前：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121115.png)

Java 7:

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121121.png)

Java 8 开始：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121132.png)

### 采用元空间替代永久代实现方法区的优点

1. 字符串存在永久代中，容易出现性能问题和内存溢出。
2. 类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出。而元空间并不在虚拟机中，而是使用本地内存，因此，默认情况下，元空间的大小仅受本地内存限制。
3. 永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。
4. 将 HotSpot 与 JRockit 合二为一。

### 方法区中的内容

#### <strong>类型信息</strong>

对每个加载的类型（类 class、接口 interface、枚举 enum、注解 annotation），JVM 必须在方法区中存储以下类型信息：

1. 这个类型的完整有效名称（即包名.类名）
2. 这个类型直接父类的完整有效名（对于 interface 或是 java. lang.Object，都没有父类）
3. 这个类型的修饰符（public、 abstract、 final 的某个子集）
4. 这个类型直接接口的一个延续列表

#### <strong>域（Field）信息</strong>

JVM 必须在方法区中保存类型的所有域的相关信息以及域的生命顺序。

域的相关信息包括：域名称、域类型、域修饰符（pubic、private、protected、static、final、volatile、transient 的某个子集）。

#### <strong>方法（method）信息</strong>

JVM 必须保存所有方法的以下信息，同域信息一样包括声明顺序：

1. 方法名称。
2. 方法的返回类型（或 void）
3. 方法参数的数量和类型（按顺序）
4. 方法的修饰符（public、private、protected、static、final、synchronized、native、abstract 的一个子集）
5. 方法的字节码（bytecodes）、操作数栈、局部变量表及大小（abstract 和 native 方法除外）
6. 异常表（abstract 和 native 方法除外），每个异常处理的开始位置、结束位置、代码处理在程序计数器中的偏移地址、被捕获的异常类的常量池索引。

#### <strong>静态变量（static 变量）</strong>

1. 静态变量和类关联在一起，随着类的加载而加载，它们成为类数据在逻辑上的一部分。
2. 类变量被类的所有实例所共享，即使没有类实例你也可以访问它。

#### <strong>全局常量（static final 变量）</strong>

与 non-final 的类变量的处理方法则不同，每个全局常量在编译的时候就被分配了。

```java
public static int count = 1;
public static final int number = 2;
```

反编译后就可以看到如下代码：

```java
public static int count;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC

public static final int number;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 2
```

## 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是《Java 虚拟机规范》中定义的内存区域。它是处于 Java 堆外的、直接向内存申请的系统内存区间。

直接内存源自 NIO，通过存在堆外内存的 DirectByteBuffer 操作 Native 本地内存。通常，访问直接内存的速度会优于 Java 堆内存，即读写性能高。因此出于性能考虑，读写频繁的场合可能考虑使用直接内存。因此 Java 的 NIO 库使用直接内存进行数据缓冲器。

直接内存也可能导致 OOM，由于它处于 Java 堆外，因此它的大小不会直接受限于 -Xmx 指定的最大堆大小，但是系统内存是有限的，Java 堆和直接内存的总和依然受限于操作系统给出的最大内存。可以通过 -XX:MaxDirectMemorySize 设置它的大小，默认与堆的最大值 -Xmx 参数值一致。

直接内存的缺点是分配回收成本较高、不受 JVM 内存回收管理。

# 常量池和 String

## 常量池概念

### 静态常量池

一个有效的 Class 文件中除了包含了类的版本信息、字段、方法以及接口等描述信息外，还包含一项信息就是<strong>常量池（Constant Pool）</strong>，用于存放编译期生成的各种<strong>字面量（Literal）</strong>和<strong>对类型、域和方法的符号引用（Symbolic References）</strong>。

### 运行时常量池

运行时常量池是方法区中的一部分。Class 文件中静态常量池的内容将在类加载后存放到方法区的运行时常量池中。也就是说，每个 Class 都有一个静态常量池，类在解析之后，将符号引用替换成直接引用，与全局常量池中的引用值保持一致。

运行时常量池相对于 Class 文件常量池的一个重要特征就是它具备动态性。运行期间也可以将新产生的常量放入常量池中，典型的实现方式就是 String 类的 intern() 方法。

### 字符串常量池

字符串的分配和其他的对象分配一样，需要耗费高昂的时间和空间为代价，如果需要大量频繁的创建字符串，会极大程度地影响程序的性能，因此 JVM 为了提高性能和减少内存开销引入了字符串常量池的概念。

#### 特点

字符串常量池中是不会存储相同的字符串的。

字符串常量池（String Pool）由 StringTable 类实现，它是一个固定大小的 Hashtable。因此，如果 String Pool 的 String 非常多，就会造成 Hash 冲突严重，从而导致链表会很长，而链表长会直接造成调用 String.intern 时性能大幅度下降。

```java
public class StringTable extends sun.jvm.hotspot.utilities.Hashtable {
    ...
}
```

JDK6、7 中，StringTable 默认是 1009，所以如果常量池中的字符串过多就会导致效率下降很快，此时可以通过参数进行调整。

JDK8 开始，StringTable 默认是 60013，也可以通过参数进行调整，但限制最小长度不低于 1009。

StringTable 存储的不是 String 对象的内容，而是一个个 HashtableEntry，HashtableEntry 里面的 value 指向的才是 String 对象一般我们说一个字符串进入了字符串常量池其实是说在这个 StringTable 中保存了对它的引用；反之，如果说没有在其中就是说 StringTable 中没有对它的引用。

#### 字符串常量池位置变化

字符串常量池自 Java 7 开始从永久代移动到堆中。因为永久代的回收频率很低，在 Full GC 的时候才会触发，而 Full GC 是老年代的空间不足、永久代不足时才会触发。这就导致了 StringTable 回收效率不高。而开发中会有大量的字符串被创建，回收效率低，导致永久代内存不足。因此，只有放到堆里才能及时回收这部分内存。

## String

无论是服务端还是 Android 客户端开发，String 对象无疑是用得最多的数据类型之一，String 类的代码简化后有如下内容：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
    
    ...
}
```

实际上，value[] 数组才是真正存放字符串的容器。

### String 对象的内存分配

以下使用 String 对象最常用的两种方式，下面进行深入分析。

#### 通过字面量赋值创建

```java
public class Test {
    public static void main(String[] args) {
        String s = "xyz";
        ...
    }
}
```

这段代码执行后（假如 main 线程还存活），内存布局如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121313png)

执行 String s = "xyz" 时，首先会在字符串常量池中查找（遍历 StringTable 逐个获取 HashtableEntry 对象，判断 HashtableEntry 的 value 是否 equals("xyz")），看能不能找到“xyz”<strong>字符串的引用</strong>，如果字符串常量池中找不到这个引用，则：

1. 创建一个 String 对象和 char 数组对象；
2. 将创建的 String 对象封装成 HashtableEntry，作为 StringTable 的 value 进行存储；
3. 返回创建的 String 对象。

如果能找到这个引用：

- 直接返回找到引用对应的 String 对象。

#### 通过 new 关键字创建

```java
public class Test {
    public static void main(String[] args) {
        String s = new String("xyz");
        ...
    }
}
```

这段代码执行后（假如 main 线程还存活），内存布局如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121321.png)

执行 String s = new String("xyz") 时，首先会在字符串常量池中查找，看能不能找到“xyz”<strong>字符串对应对象的引用</strong>，如果字符串常量池中找不到这个引用，则：

1. 创建一个 String 对象和 char 数组对象；
2. 将创建的 String 对象封装成 HashtableEntry，作为 StringTable 的 value 进行存储；
3. new String("xyz") 会在堆区又创建一个 String 对象，char 数组直接指向创建好的 char 数组对象。

如果能找到这个引用：

- new String("xyz") 会在堆区创建一个对象，char 数组直接指向已经存在的 char 数组对象。

#### 对比

对于 String s = "xyz" 这种形式创建字符串对象，如果字符串常量池中能找到，不会创建 String 对象；如果如果字符串常量池中找不到，创建一个 String 对象。

对于 String s = new String("xyz") 这种形式创建字符串对象，如果字符串常量池中能找到，创建一个 String 对象；如果如果字符串常量池中找不到，创建两个 String 对象。

所以，在日常开发中，能用 String s = "xyz" 尽量不用 String s = new String("xyz")，因为可以少创建一个对象，节省一部分空间。

事实上，运行程序用到 Test 类的时候，Test.class 文件的信息会被解析到内存的方法区里，例子中的“xyz”作为字面量，它的一个引用会被存到在堆中的字符串常量池里。而“xyz”本体还是和所有对象一样，创建在堆中的 Eden 区，但因为一直有一个引用驻留在字符串常量池，所以不会被 GC 清理掉。这个对象“xyz”会生存到整个线程结束。主线程中的 s 变量这时候都还没有被创建，但“xyz”的实例已经在堆里了，对它的引用也已经在字符串常量池里了。

有了前面的基础，再来分析下面代码：

```java
public class Test {
    public static void main(String[] args) {
        String s1 = new String("xyz");
        String s2 = "xyz";
        System.out.println(s1 == s2);
        System.out.println(s1.equals(s2));
        ...
    }
}
输出结果：
false
true
```

这段代码执行后（假如 main 线程还存活），内存布局如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121328.png)

很明显，s1 和 s2 指向的是不同的对象。而 equals 方法比较的真正的 char 数据，并且从图中可以看出 s1 和 s2 最终指向的都是同一个 char 数组对象，所以 s1.equals(s2) 等于 true。关于它们是否指向同一个数组，下图也可以证实：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121334.png)

#### 通过字符串拼接创建

```java
public class Test {
    public static void main(String[] args) {
        String s1 = "aa";
        String s2 = "bb";
        String str0 = new String("aa") + new String("bb");
        String str1 = s1 + s2;
        String str2 = "aabb";
        String str3 = "aa" + "bb";
        System.out.println(str0 == str1);
        System.out.println(str1 == str2);
        System.out.println(str2 == str3);
        ...
    }
}
输出结果：
false
false
true
```

首先，对于 str3 来说，编译期会优化成 “aabb”，因此只有它与 str2 是同一个对象。

对于 str0 和 str1 来说，会被编译器会优化成 new StringBuilder().append("aa").append("bb").toString();

StringBuilder 里面的 append 方法就是对 char 数组进行操作，查看 StringBuilder 的 toString 方法：

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    char[] value;
    
    ...

    @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
    
    ...
}
```

再看 String 的这个构造方法：

```java
public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```

它关键做了如下操作：

- 根据参数<strong>复制一份</strong> char 数组对象。
- 创建一个 String 对象，String 对象的 value 指向复制的 char 数组对象。

也就是说 str1 指向的 String 对象的 value 值直接指向了一个已经存在的数组，而且没有驻留到字符串常量池，而 str2 指向的对象驻留到字符串常量池里面去了。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121345.png)

因此 str0、str1 和 str2 都是指向不同的对象，并且分别指向不同的 char 数组对象。下图再次验证：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121351.png)

### String.intern() 方法

上面说到，调用 StringBuilder 的 toString 方法创建的 String 对象是不会驻留到字符串常量池的，那还有没有办法将这个对象驻留到字符串常量池呢？有的，这就是 String.intern 方法的作用。

以这段代码为例：

```java
String s1 = "aa";
String s2 = "bb";
String str = s1 + s2;
str.intern();
```

在执行 str.intern() 之前，内存布局如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121400.png)

在执行 str.intern();之后，内存布局如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/JVM-Memory/clipboard_20230323_121406.png)

intern 方法就是创建了一个 HashtableEntry 对象，并把 value 指向 String 对象，然后把 HashtableEntry 通过 hash 定位存到对应的字符串成常量池中。当然，前提是字符串常量池中原来没有对应的 HashtableEntry。如果字符串常量池中已经有对应的 HashtableEntry，则返回 HashtableEntry 的 value，即它所对应 String 对象的地址。

#### 版本区别

JDK6 中，将字符串对象尝试放入字符串常量池中：

如果字符串常量池中有，则不会放入，并返回已有的字符串常量池中的对象。

如果没有，会把此对象复制一份，返回字符串池常量池，并返回字符串常量池的地址。

JDK7 开始，将字符串对象尝试放入字符串池常量池中：

如果字符串常量池中有，则不会放入，并返回已有的字符串常量池中的对象。

如果没有，会把对象的引用地址复制一份，放入字符串常量池中，并返回字符串常量池中的引用地址。
