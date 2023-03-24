---
title: JVM 类加载机制
date created: 2023-03-22
date modified: 2023-03-22
tags:
  - JVM
  - 类加载机制
  - Java
---
# 概述

JVM 把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被 JVM 直接使用的 Java 类型，这个过程被称作 JVM 的类加载机制。

与那些在编译时需要进行连接的语言不同，在 Java 语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略让 Java 语言进行提前编译会面临额外的困难，也会让类加载时稍微增加一些性能开销，但是却为 Java 应用提供了极高的扩展性和灵活性，Java 天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。

# 类加载的时机

一个类型从被加载到虚拟机内存中开始到卸载位置，整个生命周期会经历如下七个阶段，其中验证、准备、解析三个部分统称为连接。
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/ClassLoader/clipboard_20230323_120031.png)
上图中，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段后再开始，这是为了支持 Java 语言的运行时半丁特性（也被称为动态绑定或者晚期绑定）。

## 主动引用

关于在什么情况下需要开始类加载过程的第一个阶段“加载”，《Java 虚拟机规范》中并没有进行强制约束，这点可以交给虚拟机的具体实现来自由把握。但是对于初始化阶段，《Java 虚拟机规范》则是严格规定了有且只有六种情况必须立即对类进行“初始化”（而加载、验证和准备阶段自然需要在此之前开始）：

- 遇到 new、getstatic、putstatic、invokestatic 这四条字节码指令时，如果类没有进行过初始化，则必须先触发其初始化。最常见的生成这四条指令的场景是：

  - 使用 new 关键字实例化对象的时候。
  - 读取或设置一个类的静态字段（被 final 修饰、已在编译期把结果放入常量池的静态字段除外）的时候。
  - 调用一个类的静态方法的时候。
- 使用 java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含 main() 方法的那个类），虚拟机会先初始化这个主类；
- 当使用 JDK 1.7 的动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果为 REF_getStatic, REF_putStatic, REF_invokeStatic 的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。
- 当一个接口中定义了 JDK 8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

## 被动引用

以上六种场景中的行为称为对一个类型进行主动引用，除此之外所有引用类型的方式都不会触发初始化，被称为被动引用。

```java
public class SuperClass {
    static {
        System.out.println("SuperClass init!");
    }

    public static int value = 123;
}

public class SubClass extends SuperClass {
    static {
        System.out.println("SubClass init!");
    }
    
    public static final String CONS = "hello";
}
```

- 通过子类引用父类的静态字段，不会导致子类初始化

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }
}
输出结果：
SuperClass init!
123
```

- 通过数组定义来引用类，不会触发此类的初始化

该过程会对数组类进行初始化，数组类是一个由虚拟机自动生成的、直接继承自 Object 的子类，其中包含了数组的属性和方法。

```java
public class Test {
    public static void main(String[] args) {
        SuperClass[] sca = new SuperClass[10];
    }
}
没有任何输出
```

- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(SubClass.CONS);
    }
}
输出结果：
hello
```

# 类加载的过程

类加载的过程包含了加载、验证、准备、解析和初始化这 5 个阶段。

## 加载

这里的加载指的是类加载的一个阶段，注意不要混淆。

加载过程完成以下三件事：

- 通过类的<strong>完全限定名称</strong>获取定义该类的二进制字节流。
- 将该字节流表示的静态存储结构转换为方法区的运行时存储结构。
- 在内存中生成一个代表该类的 Class 对象，作为方法区中该类各种数据的访问入口。

其中二进制字节流可以从以下方式中获取：

- 从 ZIP 包读取，成为 JAR、EAR、WAR 格式的基础。
- 从网络中获取，最典型的应用是 Applet。
- 运行时计算生成，例如动态代理技术，在 java.lang.reflect.Proxy 使用 ProxyGenerator.generateProxyClass 的代理类的二进制字节流。
- 由其他文件生成，例如由 JSP 文件生成对应的 Class 类。
- 从数据库中读取，这种场景相对比较少见。
- 从加密文件中获取，这是典型的防 Class 文件被反编译的保护措施，通过加载时解密 Class 文件来保障程序运行逻辑不被窥探。

## 验证

验证流程确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。验证阶段大致上会完成下面四个阶段的动作。

### 文件格式验证

主要验证字节流是否符合 Class 文件格式的规范，并且能被当前版本的虚拟机处理。

### 元数据验证

对字节码描述的信息进行语义分析，以保证其描述的信息符合《Java 语言规范》的要求。

### 字节码验证

通过数据流分析和控制流分析，确定程序语义是合法的、符合逻辑的。在第二阶段对元数据信息中的数据类型校验完毕以后，这个阶段就要对类的方法体（Class 文件中的 Code 属性）进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为。这个阶段是整个验证过程中最复杂的一个阶段。

### 符号引用验证

这个阶段的校验行为发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三个阶段——解析阶段中发生。符号引用验证可以看作是对类自身以外（常量池中的各种符号引用）的各类信息进行匹配行校验，通俗来说就是，该类是否缺少或者被禁止访问它依赖的某些外部类、方法、字段等资源。

符号引用验证的主要目的是确保解析行为能正常执行，如果无法通过符号引用验证，JVM 将会抛出异常。

## 准备

准备阶段时正式为类中定义的变量（即被 static 修饰的静态变量）<strong>分配内存并设置类变量初始值</strong>的阶段。

实例变量不会在这阶段分配内存，它会在对象实例化时随着对象一起被分配在堆中。应该注意到，实例化不是类加载的一个过程，类加载发生在所有实例化操作之前，并且类加载只进行一次，实例化可以进行多次。

类变量初始值一般为 0 值，例如下面的类变量 value 被初始化为 0 而不是 123。

public static int value = 123;

把 value 赋值为 123 的 putstatic 指令要到类的初始化阶段中执行类构造器 `<clinit>() `方法时才会被执行。

如果类变量是常量，那么它将初始化为表达式所定义的值而不是 0。例如下面的常量 value 被初始化为 123 而不是 0。

public static final int value = 123;

## 解析

解析阶段是 JVM 将常量池的符号引用替换为直接引用的过程。解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这 7 类符号引用进行。

解析过程在某些情况下可以在初始化阶段之后再开始，这是为了支持 Java 的动态绑定。

## 初始化

初始化阶段才真正开始执行类中定义的 Java 程序代码。初始化阶段是虚拟机执行类构造器 `<clinit>()` 方法的过程。在准备阶段，类变量已经赋过一次系统要求的初始零值，而在初始化阶段，则会根据程序员通过程序制定的主观计划去初始化类变量和其它资源。

- `<clinit>()` 是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序由语句在源文件中出现的顺序决定。特别注意的是，静态语句块只能访问到定义在它之前的类变量，定义在它之后的类变量只能赋值，不能访问。例如以下代码：

```java
public class Test {
    static {
        i = 0;                // 给变量赋值可以正常编译通过
        System.out.print(i);  // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

- 由于父类的 `<clinit>()` 方法先执行，也就意味着父类中定义的静态语句块的执行要优先于子类。例如以下代码：

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
     System.out.println(Sub.B);  // B 的值为 2
}
```

- `<clinit>() `方法对于类或接口来说不是必需的。
- 接口中不可以使用静态语句块，但仍然有类变量初始化的赋值操作，因此接口与类一样都会生成 `<clinit>()` 方法。但接口与类不同的是，执行接口的 `<clinit>()` 方法不需要先执行父接口的 `<clinit>()` 方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类在初始化时也一样不会执行接口的 `<clinit>()` 方法。
- 虚拟机会保证一个类的 `<clinit>()` 方法在多线程环境下被正确的加锁和同步，如果多个线程同时初始化一个类，只会有一个线程执行这个类的 `<clinit>()` 方法，其它线程都会阻塞等待，直到活动线程执行 `<clinit>()` 方法完毕。如果在一个类的 `<clinit>()` 方法中有耗时的操作，就可能造成多个线程阻塞，在实际过程中此种阻塞往往是很隐蔽的。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/ClassLoader/clipboard_20230323_120337.png)

# 类加载器

Java 虚拟机设计团队有意把了加载阶段中的“通过一个类的全限定名称来获取描述该类的二进制字节流”这个动作放到 Java 虚拟机外部去实现，以便让应用程序自己决定如何去获取所需的类。实现这个动作的代码被称为“类加载器（Class Loader）”。

## 类与类加载器

对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在 Java 虚拟机中的唯一性，每一个类加载器都拥有一个独立的类名称空间。

换句话说，虚拟句认定两个类是否相等，除了需要比较类本身相等之外，并且还要判断它们是否使用同一个类加载器进行加载，因为每一个类加载器都拥有一个独立的类名称空间。

这里的相等，包括类的 Class 对象的 equals() 方法、isAssignableFrom() 方法、isInstance() 方法的返回结果为 true，也包括使用 instanceof 关键字做对象所属关系判定结果为 true。

## 类加载器的分类

从 Java 虚拟机的角度来讲，只存在以下两种不同的类加载器：

- 启动类加载器（Bootstrap ClassLoader），使用 C++ 实现，是虚拟机自身的一部分；
- 所有其它类的加载器，使用 Java 实现，独立于虚拟机，继承自抽象类 java.lang.ClassLoader。

从 Java 开发人员的角度看，类加载器可以划分得更细致一些：

##### <strong>启动类加载器（Bootstrap ClassLoader）</strong>

此类加载器负责将存放在 <JRE_HOME>\lib 目录中的，或者被 -Xbootclasspath 参数所指定的路径中的，并且是虚拟机识别的（<strong>仅按照文件名识别</strong>，如 rt.jar，名字不符合的类库即使放在 lib 目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被 Java 程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给启动类加载器，直接使用 null 代替即可。

##### <strong>扩展类加载器（Extension ClassLoader）</strong>

这个类加载器是由 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将 <JAVA_HOME>/lib/ext 或者被 java.ext.dir 系统变量所指定路径中的所有类库加载到内存中，开发者可以直接使用扩展类加载器。

##### <strong>应用程序类加载器（Application ClassLoader）</strong>

这个类加载器是由 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。由于这个类加载器是 ClassLoader 中的 getSystemClassLoader() 方法的返回值，因此一般称为<strong>系统类加载器</strong>。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

## 双亲委派模型

应用程序是由三种类加载器互相配合从而实现类加载，除此之外还可以加入自己定义的类加载器。

下图展示了类加载器之间的层次关系，称为双亲委派模型（Parents Delegation Model）。该模型要求除了顶层的启动类加载器外，其它的类加载器都要有自己的父类加载器。这里的父子关系一般通过组合关系（Composition）来实现，而不是继承关系（Inheritance）。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/ClassLoader/clipboard_20230323_120403.png)

### 工作过程

一个类加载器收到加载请求，首先将类加载请求转发到<strong>父类加载器</strong>，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父类加载器无法完成时，子加载器才尝试自己去加载。

### 优点

1. 避免重复加载。
2. 防止 Java 核心 api 被篡改

### 实现过程

以下是抽象类 java.lang.ClassLoader 的代码片段，其中的 loadClass() 方法运行过程如下：先检查类是否已经加载过，如果没有则让父类加载器去加载。当父类加载器加载失败时抛出 ClassNotFoundException，此时尝试自己去加载。

```java
public abstract class ClassLoader {
    // The parent class loader for delegation
    private final ClassLoader parent;

    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
}

```
