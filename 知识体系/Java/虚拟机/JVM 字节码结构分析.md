---
title: JVM 字节码结构分析
comments: true
date created: 2023-03-22
date modified: 2023-03-22
id: home
layout: page
tags:
  - JVM
  - 字节码
  - Java
---

# 概述

提到字节码，首先想到的就是 Java，Java 之所以可以“一次编译，到处运行”，一是因为 JVM 针对各种操作系统、平台都进行了定制，二是因为无论在什么平台，都可以编译生成固定格式的字节码（.class 文件）供 JVM 使用。

​其实不止是 Java，其他很多编程语言如 Scala、Kotlin 和 Groovy 等都是运行在 JVM 的语言，因此它们对应的编译器也能够生成 .class 字节码。

源代码中的各种变量，关键字和运算符号的语义最终都会编译成多条字节码命令。而字节码命令所能提供的语义描述能力是要明显强于 Java 本身的，所以有其他一些同样基于 JVM 的语言能提供许多 Java 所不支持的语言特性。

![](static/boxcnkcdgXBNKvNVzhrXq8VUnRK.png)

在 Java 中一般是用 javac 命令编译源代码为字节码文件，一个 .java 文件从编译到运行的示例如下。

![](static/boxcnvjHB3lK8C1993rcvvhsAFg.png)

JVM 的指令由一个字节长度的操作码（opcode）和紧随其后的可选的操作数（operand）构成。“字节码”这个名字的由来也是因为操作码的长度用一个字节表示。

`<opcode> [<operand1>, <operand2>]`

比如将整型常量 100 压栈到栈顶的指令是 “bipush 100”，其中 bipush 就是操作码，100 就是操作数。

因为操作码长度只有 1 个字节长度，这使得编译后的字节码文件非常小巧紧凑，但同时也直接限制了整个 JVM 操作码指令集的数量最多只能有 256 个，目前已经使用了 200+。

大部分字节码指令都包含了所要操作的类型信息。比如 “ireturn” 用于返回一个 int 类型的数据，“dreturn” 用于返回一个 double 类型的的数据，“freturn” 指令用于返回一个 float 类型的数据，这种方式也使得字节码实际的指令类型远小于 200 个。

字节码使用大端序（Big-Endian）表示，即高位在前，低位在后的方式，比如字节码 “getfield 00 02”，表示的是 “getfiled 0x00<<8 | 0x02（getfield #2)”。

字节码并不是某种虚拟 CPU 的机器码，而是一种介于源码和机器码中间的一种抽象表示方法，不过字节码通过 JIT（Just in time）技术可以被进一步编译成机器码。

# 字节码结构

根据 Java 虚拟机规范，Class 文件通过 <strong>ClassFile</strong> 定义，有点类似 C 语言的结构体。

ClassFile 的结构如下：

```plain text
ClassFile {
    u4             magic; //魔数
    u2             minor_version;//小版本号
    u2             major_version;//大版本号
    u2             constant_pool_count;//常量数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//类访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口数量
    u2             interfaces[interfaces_count];//接口表
    u2             fields_count;//字段数量
    field_info     fields[fields_count];//字段表
    u2             methods_count;//方法数量
    method_info    methods[methods_count];//方法表
    u2             attributes_count;//属性数量
    attribute_info attributes[attributes_count];//属性表
}
```

可以看出，class 文件由下面十个部分组成：

- 魔数（Magic Number）
- 版本号（Minor&Major Version）
- 常量池（Constant Pool）
- 类访问标记(Access Flags)
- 类索引（This Class）
- 超类索引（Super Class）
- 接口表索引（Interfaces）
- 字段表（Fields）
- 方法表（Methods）
- 属性表（Attributes）

它们的按照以下顺序进行排放：

![](static/boxcnjjpJAE7qORzHAVoIdq9yve.png)

《Optimizing Java》的作者编了一句顺口溜帮忙记住上面这十部分："My Very Cute Animal Turns Savage In Full Moon Areas."

![](static/boxcnuIbBA41MB7NR382pr0gYQ1.png)

以下面这个类为例：

```java
public class ByteCodeDemo {
   private int a = 1;

   public int add() {
      int b = 2;
      int c = a + b;
      System.out.println(c);
      return c;
   }
}
```

编译生成的字节码如下：

```plain text
➜  Test hexdump ByteCodeDemo.class
0000000 ca fe ba be 00 00 00 34 00 24 0a 00 06 00 16 09
0000010 00 05 00 17 09 00 18 00 19 0a 00 1a 00 1b 07 00
0000020 1c 07 00 1d 01 00 01 61 01 00 01 49 01 00 06 3c
0000030 69 6e 69 74 3e 01 00 03 28 29 56 01 00 04 43 6f
0000040 64 65 01 00 0f 4c 69 6e 65 4e 75 6d 62 65 72 54
0000050 61 62 6c 65 01 00 12 4c 6f 63 61 6c 56 61 72 69
0000060 61 62 6c 65 54 61 62 6c 65 01 00 04 74 68 69 73
0000070 01 00 0e 4c 42 79 74 65 43 6f 64 65 44 65 6d 6f
0000080 3b 01 00 03 61 64 64 01 00 03 28 29 49 01 00 01
0000090 62 01 00 01 63 01 00 0a 53 6f 75 72 63 65 46 69
00000a0 6c 65 01 00 11 42 79 74 65 43 6f 64 65 44 65 6d
00000b0 6f 2e 6a 61 76 61 0c 00 09 00 0a 0c 00 07 00 08
00000c0 07 00 1e 0c 00 1f 00 20 07 00 21 0c 00 22 00 23
00000d0 01 00 0c 42 79 74 65 43 6f 64 65 44 65 6d 6f 01
00000e0 00 10 6a 61 76 61 2f 6c 61 6e 67 2f 4f 62 6a 65
00000f0 63 74 01 00 10 6a 61 76 61 2f 6c 61 6e 67 2f 53
0000100 79 73 74 65 6d 01 00 03 6f 75 74 01 00 15 4c 6a
0000110 61 76 61 2f 69 6f 2f 50 72 69 6e 74 53 74 72 65
0000120 61 6d 3b 01 00 13 6a 61 76 61 2f 69 6f 2f 50 72
0000130 69 6e 74 53 74 72 65 61 6d 01 00 07 70 72 69 6e
0000140 74 6c 6e 01 00 04 28 49 29 56 00 21 00 05 00 06
0000150 00 00 00 01 00 02 00 07 00 08 00 00 00 02 00 01
0000160 00 09 00 0a 00 01 00 0b 00 00 00 38 00 02 00 01
0000170 00 00 00 0a 2a b7 00 01 2a 04 b5 00 02 b1 00 00
0000180 00 02 00 0c 00 00 00 0a 00 02 00 00 00 01 00 04
0000190 00 02 00 0d 00 00 00 0c 00 01 00 00 00 0a 00 0e
00001a0 00 0f 00 00 00 01 00 10 00 11 00 01 00 0b 00 00
00001b0 00 5c 00 02 00 03 00 00 00 12 05 3c 2a b4 00 02
00001c0 1b 60 3d b2 00 03 1c b6 00 04 1c ac 00 00 00 02
00001d0 00 0c 00 00 00 12 00 04 00 00 00 05 00 02 00 06
00001e0 00 09 00 07 00 10 00 08 00 0d 00 00 00 20 00 03
00001f0 00 00 00 12 00 0e 00 0f 00 00 00 02 00 10 00 12
0000200 00 08 00 01 00 09 00 09 00 13 00 08 00 02 00 01
0000210 00 14 00 00 00 02 00 15
0000218
```

通过反编译后可得到如下信息：

```plain text
➜  Test javap -v ByteCodeDemo
Classfile /Users/Jie/IdeaProjects/Test/out/production/Test/ByteCodeDemo.class
  Last modified 2021年8月30日; size 536 bytes
  MD5 checksum aa05281ffe821ab9f1e1c192a69a5688
  Compiled from "ByteCodeDemo.java"
public class ByteCodeDemo
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #5                          // ByteCodeDemo
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #6.#22    // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#23    // ByteCodeDemo.a:I
   #3 = Fieldref           #24.#25   // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #26.#27   // java/io/PrintStream.println:(I)V
   #5 = Class              #28       // ByteCodeDemo
   #6 = Class              #29       // java/lang/Object
   #7 = Utf8               a
   #8 = Utf8               I
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               LByteCodeDemo;
  #16 = Utf8               add
  #17 = Utf8               ()I
  #18 = Utf8               b
  #19 = Utf8               c
  #20 = Utf8               SourceFile
  #21 = Utf8               ByteCodeDemo.java
  #22 = NameAndType        #9:#10         // "<init>":()V
  #23 = NameAndType        #7:#8          // a:I
  #24 = Class              #30            // java/lang/System
  #25 = NameAndType        #31:#32        // out:Ljava/io/PrintStream;
  #26 = Class              #33            // java/io/PrintStream
  #27 = NameAndType        #34:#35        // println:(I)V
  #28 = Utf8               ByteCodeDemo
  #29 = Utf8               java/lang/Object
  #30 = Utf8               java/lang/System
  #31 = Utf8               out
  #32 = Utf8               Ljava/io/PrintStream;
  #33 = Utf8               java/io/PrintStream
  #34 = Utf8               println
  #35 = Utf8               (I)V
{
  public ByteCodeDemo();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_1
         6: putfield      #2  // Field a:I
         9: return
      LineNumberTable:
        line 1: 0
        line 2: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  this   LByteCodeDemo;

  public int add();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_2
         1: istore_1
         2: aload_0
         3: getfield      #2 // Field a:I
         6: iload_1
         7: iadd
         8: istore_2
         9: getstatic     #3 // Field java/lang/System.out:Ljava/io/PrintStream;
        12: iload_2
        13: invokevirtual #4 // Method java/io/PrintStream.println:(I)V
        16: iload_2
        17: ireturn
      LineNumberTable:
        line 5: 0
        line 6: 2
        line 7: 9
        line 8: 16
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      18     0  this   LByteCodeDemo;
            2      16     1     b   I
            9       9     2     c   I
}
SourceFile: "ByteCodeDemo.java"
```

反编译的过程将看似毫无规律的十六进制数字还原出了更直观易懂的信息，可见字节码的生成必然遵循着某个固定而特殊的规律，下面将一一对它们进行解析。

## 魔数（Magic Number）

.class 文件的头四个字节称为魔数（Magic Number），可以看到 .class 的魔数为 0xCAFEBABE。很多文件都以魔数来进行文件类型的区分，比如 PDF 文件的魔数是 %PDF-(16 进制 0x255044462D)，png 文件的魔数是\x89PNG（0x89504E47）。文件格式的制定者可以自由的选择魔数值，只要魔数值还没有被广泛的采用过且不会引起混淆即可。

魔数放在文件开头，JVM 可以根据文件的开头来判断这个文件是否可能是一个 .class 文件，如果是，才会继续进行之后的操作。

## 版本号（Minor & Major Version）

版本号为魔数之后的 4 个字节，前两个字节表示次版本号（Minor Version），后两个字节表示主版本号（Major Version）。上面示例中版本号对应的字节码为“00 00 00 34”，次版本号转化为十进制为 0，主版本号转化为十进制为 52，对应的主版本号为 1.8，所以编译该文件的 Java 版本号为 1.8.0（每次 Java 发布大版本时，主版本号加 1）。

## 常量池（Constant Pool）

紧接着主版本号之后的字节为常量池入口。常量池整体上分为两部分：常量池计数器（存储常量池大小计数）以及常量池数据区（存储常量池项），如下图所示。

![](static/boxcnRAcBVKioatXxFhpOOr3F9e.png)

### 常量池计数器（constant_pool_count）

由于常量的数量不固定，所以需要先放置两个字节来表示常量池容量计数值。本文示例中的代码常量池计数对应的字节码为“00 24”，将十六进制的 24 转化为十进制值为 36，排除掉下标“0”，也就是说，这个类文件中共有 35 个常量。

### 常量池数据区（constant_pool[constant_pool_count-1]）

常量池项中存储两类常量：字面量与符号引用。字面量如声明为 Final 的常量值和文本字符串等，符号引用分为类和接口的全局限定名、字段的名称和描述符、方法的名称和描述符。

数据区是由（constant_pool_count-1）个 cp_info 结构组成，一个 cp_info 结构对应一个常量。

目前在字节码中共有 14 种类型的 cp_info（如下图所示），每种类型的结构都是固定的。

| 类型                             | 标志（tag） | 描述                 |
| -------------------------------- | ----------- | -------------------- |
| CONSTANT_utf8_info               | 1           | UTF-8 编码的字符串   |
| CONSTANT_Integer_info            | 3           | 整形字面量           |
| CONSTANT_Float_info              | 4           | 浮点型字面量         |
| CONSTANT_Long_info               | ５          | 长整型字面量         |
| CONSTANT_Double_info             | ６          | 双精度浮点型字面量   |
| CONSTANT_Class_info              | ７          | 类或接口的符号引用   |
| CONSTANT_String_info             | ８          | 字符串类型字面量     |
| CONSTANT_Fieldref_info           | ９          | 字段的符号引用       |
| CONSTANT_Methodref_info          | 10          | 类中方法的符号引用   |
| CONSTANT_InterfaceMethodref_info | 11          | 接口中方法的符号引用 |
| CONSTANT_NameAndType_info        | 12          | 字段或方法的符号引用 |
| CONSTANT_MothodType_info         | 16          | 方法类型             |
| CONSTANT_MethodHandle_info       | 15          | 方法句柄             |
| CONSTANT_InvokeDynamic_info      | 18          | 动态方法调用点       |

具体以 CONSTANT_utf8_info 为例，它的结构如下图所示。首先一个字节“tag”，它的值取自上图中对应项的 Tag，由于它的类型是 utf8_info，所以值为“01”。接下来两个字节标识该字符串的长度 Length，然后 Length 个字节为这个字符串具体的值。

从前文示例中的字节码摘取一个 cp_info 结构，如下图所示。

![](static/boxcnlJJ3NIsuvEuTTWfzB3gJWb.png)

将它翻译过来后，其含义为：该常量类型为 utf8 字符串，长度为一字节，数据为“a”。

![](static/boxcnoYCKARu6A7fWgD6J8y5OOJ.png)

下面来逐一详细介绍这些常量池项。

#### CONSTANT_Utf8_info

CONSTANT_Utf8_info 存储的是经过 MUTF-8(modified UTF-8) 编码的字符串，结构如下:

```plain text
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

它由三部分构成：

1. 第一部分 tag 值为固定为 1
2. 第二部分 length 表示字符串的长度
3. 第三部分是采用 MUTF-8 编码的长度为 length 的字节数组

如果要存储的字符串是"hello"，存储结构如下图所示

![](static/boxcnglozJXM2xsd4b9XtXKtMXd.png)

MUTF-8 编码与标准的 UTF-8 编码大部分情况下是相同的，但也有一些细微的区别，比如在 MUTF-8 里空字符("\u0000")用两个字节 0xC080 表示，而在标准的 UTF-8 编码里的表述方式为 0x00，还有一些其它的差异这里不做深入的展开。

#### CONSTANT_Integer_info 和 CONSTANT_Float_info

这两种结构分别用来表示 int 和 float 类型的常量，这两种类型的结构很类似，都用四个字节来表示具体的数值常量，它们的结构定义如下：

```plain text
CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
}
CONSTANT_Float_info {
    u1 tag;
    u4 bytes;
}
```

以整型常量 18(0x12) 为例，它在常量池中的布局结构如下图所示：

![](static/boxcnQxx1hRx7YZ5WTY33jM9Jkd.png)

Java 语言规范还定义了 boolean、byte、short 和 char 类型的变量，在常量池中都会被当做 int 来处理，比如用代码定义了下面的常量：

```plain text
public class MyConstantTest {
    public final boolean bool = true; //  1(0x01)
    public final char c = 'A';        // 65(0x41)
    public final byte b = 66;         // 66(0x42)
    public final short s = 67;        // 67(0x43)
    public final int i = 68;          // 68(0x44)
}
```

编译生成的 class 文件如下图所示：

![](static/boxcnxghxz55lxR24BEeO7J4IFd.png)

#### CONSTANT_Long_info 和 CONSTANT_Double_info

这两种结构分别用来表示 long 和 double 类型的常量，这两个结构类似，都用 8 个字节表示具体的常量数值。它们的结构如下：

```plain text
CONSTANT_Long_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}

CONSTANT_Double_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
```

对应的结构如下图所示：

![](static/boxcnw4fKxeJeUtoutX0LlsH9xd.png)

以下面的代码为例

```plain text
public class HelloWorldMain {
    public final long a = Long.MAX_VALUE; 
}
```

javap 输出的常量池信息如下：

```plain text
Constant pool:
   #1 = Methodref          #7.#17         // java/lang/Object."<init>":()V
   #2 = Class              #18            // java/lang/Long
   #3 = Long               9223372036854775807l
   #5 = Fieldref           #6.#19         // HelloWorldMain.a:J
```

前面提到过，CONSTANT_Long_info 和 CONSTANT_Double_info 占用两个常量池位置，可以看到常量 a 占用了 #3 和 #4 两个位置，下一个常量从索引值 5 开始。

![](static/boxcnYdwhTSIEh2LOVH65Q9Roed.png)

#### CONSTANT_Class_info

CONSTANT_Class_info 结构用来表示类或接口，它的结构如下：

```plain text
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
```

它由两部分组成，第一个字节是 tag，值为 7，tag 后面的两个字节 name_index 是一个常量池索引，指向类型为 CONSTANT_Utf8_info 常量，这个字符串存储的是类或接口的全限定名。如下图所示：

![](static/boxcnsbv5Pcz2WFCdEy1irInLdc.png)

#### CONSTANT_String_info

CONSTANT_String_info 用来表示 java.lang.String 类型的常量对象，同样由两部分构成：

```plain text
CONSTANT_String_info {
    u1 tag;
    u2 string_index;
}
```

第一个字节是 tag，值为 8，tag 后面的两个字节是一个叫 string_index 的索引值，指向常量池中的 CONSTANT_Utf8_info，这个 CONSTANT_Utf8_info 中存储的才是真正的字符串常量。

以下面的代码为例：

```java
public class HelloWorldMain {
    private String a = "hello";
}
```

CONSTANT_String_info 的存储布局方式为：

![](static/boxcnkuaAMiOGNfEanZiRNtV1dh.png)

#### CONSTANT_Fieldref_info、CONSTANT_Methodref_info 和 CONSTANT_InterfaceMethodref_info

这三种常量类型结构比较类似

```plain text
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

下面以 CONSTANT_Methodref_info 为例来介绍如何描述一个方法。

方法 = 方法所属的类 + 方法名 + 方法参数和返回值描述符

这就是 CONSTANT_Methodref_info 的作用，它表示类中方法的符号引用

它由三部分构成

- 第一部分也是 tag，值为 10
- 第二个部分是 class_index，是一个指向 CONSTANT_Class_info 的常量池索引值
- 第三部分是 name_and_type_index，是一个指向 CONSTANT_NameAndType_info 的常量池索引值，表示方法的参数类型和返回值的签名

```java
public class HelloWorldMain {
    public static void main(String[] args) {
        new HelloWorldMain().testMethod(1, "hi");
    }
    public void testMethod(int id, String name) {
    }
}
```

Constant pool:

```plain text
   #2 = Class              #18            // HelloWorldMain
   #5 = Methodref          #2.#20         // HelloWorldMain.testMethod:(ILjava/lang/String;)V
  #20 = NameAndType        #13:#14        // testMethod:(ILjava/lang/String;)V
```

结构布局示意图如下：

![](static/boxcnX4C4I8q1iJ5YNvckikrLXg.png)

#### CONSTANT_NameAndType_info

CONSTANT_NameAndType_info 结构用来表示字段或者方法，格式如下：

```plain text
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```

它由三部分组成：

- tag : 值为 12
- name_index : 指向常量池中的 CONSTANT_Utf8_info，存储的是字段名或者方法名
- descriptor_index : 也是指向常量池中的 CONSTANT_Utf8_info，存储的是字段描述符或者方法描述符

以下面的方法为例：

```java
public void testMethod(int id, String name) {
}
```

CONSTANT_NameAndType_info 的结构布局示意图如下：

![](static/boxcnTMVJQxDbRQr2Yj56r48Ywd.png)

#### CONSTANT_MethodType_info、CONSTANT_MethodHandle_info 和 CONSTANT_InvokeDynamic_info

从 JDK1.7 开始，为了更好的支持动态语言调用，新增了以上 3 种常量池类型。

以 CONSTANT_InvokeDynamic_info 为例，它主要为 invokedynamic 指令提供启动引导方法，它由三部分构成：

```plain text
CONSTANT_InvokeDynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```

- tag：值为 18
- bootstrap_method_attr_index：指向引导方法表 bootstrap_methods[] 数组的索引
- name_and_type_index：指向索引类常量池里 CONSTANT_NameAndType_info，表示方法描述符

比如下面的的代码：

```java
public void foo() {
    new Thread (()-> {
        System.out.println("hello");
    }).start();
}
```

javap 输出的常量池的部分如下：

```plain text
Constant pool:
   #3 = InvokeDynamic      #0:#25         // #0:run:()Ljava/lang/Runnable;
   ...
  #25 = NameAndType        #37:#38        // run:()Ljava/lang/Runnable;

BootstrapMethods:
  0: #22 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #23 ()V
      #24 invokestatic HelloWorldMain.lambda$foo$0:()V
      #23 ()V
```

整体的结构如下图所示：

![](static/boxcnRDdo2x0Kya4uGFXaka6gNe.png)

## 类访问标记(Access Flags)

常量池之后存储的是访问标记（Access flags），用来标识一个类是是不是 final、abstract 等，由两个字节表示总共可以有 16 个标记位可供使用，目前只使用了其中的 8 个。

![](static/boxcnIlKqPR8zZnaVqJsjZMEYNg.png)

具体的标记位含义如下

| Flag Name      | Value | Interpretation                         |
| -------------- | ----- | -------------------------------------- |
| ACC_PUBLIC     | 1     | 标识是否是 public                      |
| ACC_FINAL      | 10    | 标识是否是 final                       |
| ACC_SUPER      | 20    | 已经不用了                             |
| ACC_INTERFACE  | 200   | 标识是类还是接口                       |
| ACC_ABSTRACT   | 400   | 标识是否是 abstract                    |
| ACC_SYNTHETIC  | 1000  | 编译器自动生成，不是用户源代码编译生成 |
| ACC_ANNOTATION | 2000  | 标识是否是注解类                       |
| ACC_ENUM       | 4000  | 标识是否是枚举类                       |

JVM 并没有穷举所有的访问标志，而是使用按位或操作来进行描述的，比如某个类的修饰符为 Public Final，则对应的访问修饰符的值为 ACC_PUBLIC | ACC_FINAL，即 0x0001 | 0x0010=0x0011。

## 类索引（This Class）、超类索引（Super Class）和接口表索引（Interfaces）

这三个部分用来确定类的继承关系，this_class 表示类索引，super_name 表示父类索引，interfaces 表示类或者接口的直接父接口。以 this_class 为例，它是一个两字节组成，执行常量池。

以下面代码为例：

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World");
    }
}
```

this_class 为 0x0005，指向常量池中下标为 5 的元素，这个元素是由两部分组成，第一部分是类型，这里是 Class 表示是一个类，第二部分是指向常量池下标 21 的元素，这个元素是字符串 "HelloWorldMain"。

![](static/boxcnYfH6pokcjiBINJXkypd1Wh.png)

super_class 和 interfaces 的原理与之类似。

## 字段表（Fields）

### 字段表概述

紧随接口索引表之后的是字段表（Fields），类中定义的字段会被存储到这个集合中，包括类中定义的静态和非静态的字段，不包括方法内部定义的变量。

结构如下:

```java
{
    u2             fields_count;
    field_info     fields[fields_count];
}
```

由两部分组成

- 字段数量（fields_count）：字段表也是一个变长的结构，类中定义的若干个字段的个数会被存储到字段数量里。
- 字段集合（fields）：字段集合是一个类数组的结构，共有 fields_count 个，对应类中定义的若干个字段，每一个字段 field_info 的结构会在下面介绍。

如下图所示:

![](static/boxcneFvQOLATDfINYhTn60HaNp.png)

### 字段结构

每个字段（field_info）的格式如下：

```plain text
field_info {
    u2             access_flags; 
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

字段结构分为四部分：

- access_flags：字段的访问标记。是 public、private 还是 protected，是否是 static，是否是 final 等。
- name_index：字段名的索引值，指向常量池的的字符串常量。
- descriptor_index：字段描述符的索引，指向常量池的字符串常量。
- attributes_count、attribute_info：属性的个数和属性集合。

这些组成部分接下来的会有详细介绍。

### 字段访问标记

与类一样，字段也拥有自己的字段访问标记，不过要比类的访问标记要更丰富一些，共有 9 种，详细的列表如下：

| 访问标记名    | 十六进制值 | 描述                                                      |
| ------------- | ---------- | --------------------------------------------------------- |
| ACC_PUBLIC    | 0x0001     | 声明为 public                                             |
| ACC_PRIVATE   | 0x0002     | 声明为 private                                            |
| ACC_PROTECTED | 0x0004     | 声明为 protected                                          |
| ACC_STATIC    | 0x0008     | 声明为 static                                             |
| ACC_FINAL     | 0x0010     | 声明为 final                                              |
| ACC_VOLATILE  | 0x0040     | 声明为 volatile，解决内存可见性的问题                     |
| ACC_TRANSIENT | 0x0080     | 声明为 transient，被 transient 修饰的字段默认不会被序列化 |
| ACC_SYNTHETIC | 0x1000     | 表示这个字段是由编译器自动生成，而不是用户代码编译产生    |
| ACC_ENUM      | 0x4000     | 表示这是一个枚举类型的变量                                |

如果在类中定义了如下的字段：

```java
public static final int DEFAULT_SIZE = 128;
```

编译后 DEFAULT_SIZE 字段在类文件中存储的访问标记值为 0x0019，则它的访问标记为 ACC_PUBLIC | ACC_STATIC | ACC_FINAL，表示它是一个 public static final 类型的变量。如下图所示：

![](static/boxcnfiGV3FwMsuDmrPfKGeyFsh.png)

### 字段描述符

当定义一个 int 类型的变量时，类文件中存储的类型并不是字符串的 “int”，而是使用了更精简的 I 来表示。

<strong>引用类型</strong>使用 “L;” 的方式来表示为了防止多个连续的引用类型描述符出现混淆，引用类型描述符最后都加上了一个 “;” 作为分隔。比如字符串类型 String 的描述符为  “Ljava/lang/String;"，

JVM 使用一个前置的 “[” 来表示数组类型，如 int[] 类型描述符为 “[I”，字符串数组 String[] 的描述符为 “[Ljava/lang/String;”。多维数组描述符只是多加了几个 “[” 而已，比如 Object[][][] 类型描述符为 “[[[Ljava/lang/Object;”。

完整的字段类型描述符映射表如下所示：

| 描述符        | 类型                                     |
| ------------- | ---------------------------------------- |
| B             | byte 类型                                |
| C             | char 类型                                |
| D             | double 类型                              |
| F             | float 类型                               |
| I             | int 类型                                 |
| J             | long 类型                                |
| S             | short 类型                               |
| Z             | bool 类型                                |
| L ClassName ; | 引用类型，"L" + 对象类型的全限定名 + ";" |
| [             | 一维数组                                 |

### 字段属性表

与字段相关的属性有下面这几个：ConstantValue、Synthetic 、Signature、Deprecated、RuntimeVisibleAnnotations 和 RuntimeInvisibleAnnotations 这六个，比较常见的是 ConstantValue 这属性，用来表示一个常量字段的值。

### 案例分析

以文章开头 ByteCodeDemo 类的字节码中的字段表为例，如下图所示。

![](static/boxcnhkYCPUuhnfj6seMHJCkV5L.png)

0001：字段表计数。实例代码中只有一个字段，故值为 1

剩下为字段属性，一共由四个部分组成：

1. 0002：表示字段的访问标志。对应为 Private
2. 0007：字段名称。指向常量池中第 7 项：字符 “a”
3. 0008：字段描述符。指向常量池中第 8 项：字符 “I”（代表 int）
4. 0000：字段属性个数。没有属性，值为 0

综上，就可以唯一确定出一个类中声明的变量：private int a。

## 方法表（Methods）

### 方法表概述

在字段表后面的是方法表，类中定义的方法会被存储在这里，与前面介绍的字段表很类似，方法表也是一个变长结构：

```java
{
    u2             methods_count;
    method_info    methods[methods_count];
}
```

由表示方法个数的 methods_count 和对应个数的方法项集合组成，如下图所示：

![](static/boxcnYRn6VyBjUALNLwpaaXmCGh.png)

### 方法结构

每个字段（method_info）的格式如下：

```plain text
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

字段结构分为四部分：

- access_flags：方法的访问标记。是 public、private 还是 protected，是否是 static，是否是 final 等。
- name_index：方法名的索引值，指向常量池的的字符串常量。
- descriptor_index：方法描述符的索引，指向常量池的字符串常量。
- attributes_count、attribute_info：属性的个数和属性集合。

### 方法访问标记

方法的访问标记比类和字段的访问标记类型更丰富，有 12 种之多，如下表所示：

| 方法访问标记     | 值     | 描述                                                          |
| ---------------- | ------ | ------------------------------------------------------------- |
| ACC_PUBLIC       | 0x0001 | 声明为 public                                                 |
| ACC_PRIVATE      | 0x0002 | 声明为 private                                                |
| ACC_PROTECTED    | 0x0004 | 声明为 protected                                              |
| ACC_STATIC       | 0x0008 | 声明为 static                                                 |
| ACC_FINAL        | 0x0010 | 声明为 final                                                  |
| ACC_SYNCHRONIZED | 0x0020 | 声明为 synchronized                                           |
| ACC_BRIDGE       | 0x0040 | bridge 方法, 由编译器生成                                     |
| ACC_VARARGS      | 0x0080 | 方法包含可变长度参数，比如 String... args                     |
| ACC_NATIVE       | 0x0100 | 声明为 native                                                 |
| ACC_ABSTRACT     | 0x0400 | 声明为 abstract                                               |
| ACC_STRICT       | 0x0800 | 声明为 strictfp，表示使用 IEEE-754 规范的精确浮点数，极少使用 |
| ACC_SYNTHETIC    | 0x1000 | 表示这个方法是由编译器自动生成，而不是用户代码编译产生        |

以下面的代码为例：

```java
private static synchronized void foo() {
}
```

生成的类文件中 foo 方法的访问标记等于 0x002a（ACC_PRIVATE | ACC_STATIC | ACC_SYNCHRONIZED），，表示这是一个 private static synchronized 的方法，如下图所示：

![](static/boxcn5MLe8x4tSSIltG15Zix6Bb.png)

### 方法名与描述符

紧随方法访问标记是方法名索引 name_index，指向常量池中 CONSTANT_Utf8_info 类型的字符串常量，比如有这样一个方法定义 “private void foo()”，编译器会生成一个类型为 CONSTANT_Utf8_info 的字符串常量项，里面存储了 “foo”，方法名索引 name_index 指向了这个常量项。

方法描述符索引 descriptor_index，它也是方法名指向常量池中类型为 CONSTANT_Utf8_info 的字符串常量项。方法描述符用来表示一个方法所需参数和返回值，格式为：

(参数 1 类型 参数 2 类型 参数 3 类型 ...)返回值类型

比如方法 “Object foo(int i, double d, Thread t)” 的描述符为“(IDLjava/lang/Thread;)Ljava/lang/Object;”，其中 “I” 表示第一个参数 i 的参数类型 int，“D” 表示第二个参数 d 的类型 double，“Ljava/lang/Thread;” 表示第三个参数 t 的类型 Thread，“Ljava/lang/Object;” 表示返回值类型 Object。

如下图所示:

![](static/boxcnohqABY9UKJFDi2ic5S6pyb.png)

### 方法属性表

前面介绍了方法的访问标记、方法签名，还有一些重要的信息没有出现，如方法声明抛出的异常，方法的字节码，方法是否被标记为 deprecated，这些信息存在哪里呢？这就是方法属性表的作用。跟方法相关的属性有很多，其中重要的是 Code 和 Exceptions 属性，其中 Code 属性存放方法体的字节码指令，Exceptions 属性用于存储方法声明抛出的异常。

### 案例分析

还是以文章中 ByteCodeDemo 类的字节码中的字段表为例，如下图所示。

![](static/boxcn2QIufOsSvDNcqrr73NMJYe.png)

共分为四个部分：

1. 0001：方法访问标记。表示 public
2. 0010：方法名的索引值。指向第 16 个常量，即 “add”
3. 0011：方法描述符的索引。指向第 17 个常量，即 ()I（返回值类型为 int）
4. 0001000b...：属性表。0001 表示只有一个属性，000b 表示属性名称为 “Code”，具体分析见下文

## 属性表（Attributes）

### 属性表概述

在方法表之后的结构是 class 文件的最后一步部分属性表。<strong>属性出现的地方比较广泛，不止出现在字段和方法中，在 class 文件中也会出现。</strong>

相比于常量池只有 14 种固定的类型，属性表是的类型是更加灵活的，不同的虚拟机实现厂商可以自定义自己的属性，JVM 运行时会忽略掉它不认识的属性。

属性表的结构如下所示：

```java
{
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

与其它结构类似，属性表使用两个字节表示属性的个数 attributes_count，接下来是若干个属性项的集合，可以看做是一个数组，数组的每一项都是一个属性项 attribute_info，数组的大小为 attributes_count。

### 属性结构

每个属性（attribute_info）的格式如下：

```plain text
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```

属性结构分为三部分：

- attribute_name_index：属性名索引
- attribute_length：属性长度
- info[attribute_length]：属性数组

属性表其实只是定义了属性的长度，里面还有一个不定长的数组，具体的结构需要自己定义。

### 属性类型

| 属性名称                            | 使用位置           | 含义                                                                                              |
| ----------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------- |
| Code                                | 方法表             | Java 代码编译成的字节码指令                                                                       |
| ConstantValue                       | 字段表             | final 关键字定义的常量池                                                                          |
| Deprecated                          | 类，方法表，字段表 | 被声明为 deprecated 的方法和字段                                                                  |
| Exceptions                          | 方法表             | 方法抛出的异常                                                                                    |
| EnclosingMethod                     | 类文件             | 仅当一个类为局部类或者匿名类是才能拥有这个属性，这个属性用于标识这个类所在的外围方法              |
| InnerClass                          | 类文件             | 内部类列表                                                                                        |
| LineNumberTable                     | Code 属性          | Java 源码的行号与字节码指令的对应关系                                                             |
| LocalVariableTable                  | Code 属性          | 方法的局部变量描述                                                                                |
| StackMapTable                       | Code 属性          | JDK1.6 中新增的属性，供新的类型检查检验器检查和处理目标方法的局部变量和操作数有所需要的类是否匹配 |
| Signature                           | 类，方法表，字段表 | 用于支持泛型情况下的方法签名                                                                      |
| SourceFile                          | 类文件             | 记录源文件名称                                                                                    |
| SourceDebugExtension                | 类文件             | 用于存储额外的调试信息                                                                            |
| Synthetic                           | 类，方法表，字段表 | 标志方法或字段为编译器自动生成的                                                                  |
| LocalVariableTypeTable              | 类                 | 使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加                          |
| RuntimeVisibleAnnotations           | 类，方法表，字段表 | 为动态注解提供支持                                                                                |
| RuntimeInvisibleAnnotations         | 表，方法表，字段表 | 用于指明哪些注解是运行时不可见的                                                                  |
| RuntimeVisibleParameterAnnotation   | 方法表             | 作用与 RuntimeVisibleAnnotations 属性类似，只不过作用对象为方法                                   |
| RuntimeInvisibleParameterAnnotation | 方法表             | 作用与 RuntimeInvisibleAnnotations 属性类似，作用对象哪个为方法参数                               |
| AnnotationDefault                   | 方法表             | 用于记录注解类元素的默认值                                                                        |
| BootstrapMethods                    | 类文件             | 用于保存 invokeddynamic 指令引用的引导方式限定符                                                  |

### ConstantValue 属性

ConstantValue 属性只会出现字段 field_info 中，表示静态变量的初始值，它的结构如下：

```plain text
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
```

其中 attribute_name_index 是指向常量池中值为 "ConstantValue" 的常量项，ConstantValue 属性的 attribute_length 值恒定为 2，constantvalue_index 指向常量池中具体的常量值索引，根据变量的类型不同 constantvalue_index 指向不同的常量项。

以 “public static final int DEFAULT_SIZE = 128;” 为例，字段对应的 class 文件如下图高亮部分：

![](static/boxcnrzZizdz4XgrKQI7exQWeSf.png)

它对应的字段结构如下：

![](static/boxcnI5pQuvpSf8UwTkp6TOq8xf.png)

### Code 属性

Code 属性可以说是类文件中最重要的组成部分了，它包含了所有方法的字节码，结构如下：

```plain text
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

Code 属性表的字段含义如下：

- 属性名索引（attribute_name_index）占两个字节，指向常量池中 CONSTANT_Utf8_info 常量，表示属性的名字，比如这里对应的常量池的字符串常量"Code"。
- 属性长度（attribute_length）占用两个字节，表示属性值大小。
- max_stack 表示操作数栈的最大深度，方法执行的任意期间操作数栈的深度都不会超过这个值。它的计算规则是有入栈的指令 stack 增加，有出栈的指令 stack 减少，在整个过程中 stack 的最大值就是 max_stack 的值，增加和减少的值一般都是 1，但也有例外：LONG 和 DOUBLE 相关的指令入栈 stack 会增加 2，VOID 相关的指令则为 0。
- max_locals 表示局部变量表的大小，它的值并不是等于方法中所有局部变量的数量之和。当一个局部作用域结束，它内部的局部变量占用的位置就可以被接下来的局部变量复用了。
- code_length 和 code 用来表示字节码相关的信息，其中 code_length 表示字节码指令的长度，占用 4 个字节。code 是一个长度为 code_length 的字节数组，存储真正的字节码指令。
- exception_table_length 和 exception_table 用来表示代码内部的异常表信息，如我们熟知的 try-catch 语法就会生成对应的异常表。
- attributes_count 和 attributes[] 用来表示 Code 属性相关的附属属性，Java 虚拟机规定 Code 属性只能包含这四种可选属性：LineNumberTable、LocalVariableTable、LocalVariableTypeTable、StackMapTable。以 LineNumberTable 为例，LineNumberTable 用来存放源码行号和字节码偏移量之间的对应关系，这 LineNumberTable 属于调试信息，不是类文件运行的必需的属性，默认情况下都会生成。如果没有这两个属性，那么在调试时没有办法在源码中设置断点，也没有办法在代码抛出异常的时候在错误堆栈中显示出错的行号信息。

以下面的代码为例：

```java
public static void main(String[] args) {
    try {
        foo();
    } catch (NullPointerException e) {
        System.out.println();
    } catch (IOException e) {
        System.out.println();
    }

    try {
        foo();
    } catch (Exception e) {
        System.out.println();
    }
}
```

如下图所示：

![](static/boxcn8Vve3ec7bNJlvplJIhPbSg.png)

### 案例分析

同样以 ByteCodeDemo 类的字节码为例，如下图所示。

![](static/boxcnTyhgjV3Q99Kdh9Zn59r0Dc.png)

0001 表示为属性表计数，这里只有一个附加属性。属性表

```plain text
SourceFile_attribute {
     u2 attribute_name_index;
     u4 attribute_length;
     u2 sourcefile_index;
}
```

SourceFile_attribute 一共分为三个部分。

0014：指向第 20 个常量，为 “SourceFile”，说明这个属性是 Source

00000002：表示长度为 2

0015：指向第 21 个常量，即 “ByteCodeDemo.java”
