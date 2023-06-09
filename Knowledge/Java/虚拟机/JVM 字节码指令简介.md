---
title: JVM 字节码指令简介
date created: 2023-03-22
date modified: 2023-03-22
tags:
  - JVM
  - 字节码
  - Java
---
Java 虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作码）以及跟随其后的零至多个代表此操作所需的参数（操作数）构成。

在 Java 虚拟机的指令集中，大多数指令都包含其操作所对应的数据类型信息。比如， iload 指令用于从局部变量表中加载 int 型的数据到操作数栈中，而 fload 指令加载的则是 float 类型的数据。这两条指令的操作在虚拟机内部可能会是由同一段代码来实现的，但在 Class 文件中它们必须拥有各自独立的操作码。

编译器会在编译期或运行期将 byte 和 short 类型的数据带符号扩展为相应的 int 类型数据，将 boolean 和 char 类型数据零位扩展为相应的 int 类型数据。因此，大多数对于 boolean、byte、short 和 char 类型数据的操作，实际上都是使用相应的对 int 类型作为运算类型来进行的。

### 加载和存储指令

加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈之间来回传输。这些指令如下：

- 将一个局部变量加载到操作栈:iload、iload_、lload、lload_、fload、fload_、dload、dload_、aload、aload_
- 将一个数值从操作数栈存储到局部变量表:istore、istore_、lstore、lstore_、fstore、fstore_、dstore、dstore_、astore、astore_
- 将一个常量加载到操作数栈:bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、iconst_m1、iconst_、lconst_、fconst_、dconst_
- 扩充局部变量表的访问索引的指令:wide

### 运算指令

算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。

- 加法指令:iadd、ladd、fadd、dadd
- 减法指令:isub、lsub、fsub、dsub
- 乘法指令:imul、lmul、fmul、dmul
- 除法指令:idiv、ldiv、fdiv、ddiv
- 求余指令:irem、lrem、frem、drem
- 取反指令:ineg、lneg、fneg、dneg
- 位移指令:ishl、ishr、iushr、lshl、lshr、lushr
- 按位或指令:ior、lor ·按位与指令:iand、land
- 按位异或指令:ixor、lxor ·局部变量自增指令:iinc
- 比较指令:dcmp g、dcmp l、fcmp g、fcmp l、lcmp

### 类型转换指令

类型转换指令可以将两种不同的数值类型相互转换，这些转换操作一般用于实现用户代码的显示类型转换操作，或者用来处理字节码指令集中数据类型相关指令无法与数据类型一一对应的问题。

Java 虚拟机直接支持以下数值类型的宽化类型转换：

- int 类型到 long、float 或者 double 类型
- long 类型到 float、double 类型
- float 类型到 double 类型

相对的，处理窄化类型转换时，就必须显示地使用转换指令来完成：i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l 和 d2f

在将 int 或 long 类型窄化转换为整数类型 T 的时候，转换过程仅仅是简单丢弃除最低位 N 字节以外的内容，N 是类型 T 的数据类型长度，这将可能导致转换结果与输入值有不同的正负号（把前面的符号位给舍去了）。

### 对象创建与访问指令

虽然类实例和数组都是对象，但 Java 虚拟机对类实例和数组的创建与操作使用了不同的字节码指令。对象创建后，就可以通过对象访问指令获取对象实例或者数组实例中的字段或者数组元素。

- 创建类实例的指令:new
- 创建数组的指令:new array 、anew array 、mult ianew array
- 访问类字段(static 字段，或者称为类变量)和实例字段(非 static 字段，或者称为实例变量)的 指令:getfield、putfield、getstatic、putstatic
- 把一个数组元素加载到操作数栈的指令:baload、caload、saload、iaload、laload、faload、 daload、aaload
- 将一个操作数栈的值储存到数组元素中的指令:bastore、castore、sastore、iastore、fastore、 dastore、aastore
- 取数组长度的指令:array lengt h - 检查类实例类型的指令:inst anceof、checkcast

### 操作数栈管理指令

如同操作一个普通数据结构中的堆栈那样，Java 虚拟机提供了一些用于直接操作操作数栈的指令：

- 将操作数栈的栈顶一个或两个元素出栈:p op 、p op 2
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶:dup 、dup 2、dup _x1、 dup 2_x1、dup _x2、dup 2_x2
- 将栈最顶端的两个数值互换:swap

### 控制转移指令

控制转移指令可以让 Java 虚拟机有条件或无条件地从指定位置指令(而不是控制转移指令)的下一条指令继续执行程序，从概念模型上理解，可以认为控制指令就是在有条件或无条件地修改 PC 寄存器的值。

- 条件分支:ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq 和 if_acmpne
- 复合条件分支:tableswitch、lookupswitch
- 无条件分支:goto、goto_w、jsr、jsr_w、ret

与前面的算术运算的规则一致，对于 boolean、byte、char 和 short 类型的条件分支比较操作，都使用 int 类型的比较指令完成，而对于 long、float 和 double 类型的条件分支比较操作，则会先执行相应类型的比较运算指令，预算指令会返回一个整型值到操作数栈中，随后再执行 int 类型的条件分支比较操作来完成整个分支跳转。因此，各种类型的比较最终都会转换为 int 类型的比较操作。

### 方法调用和返回指令

方法调用：分派、执行过程。

- invokevirt ual 指令:用于调用对象的实例方法，根据对象的实际类型进行分派(虚方法分派)， 这也是 Java 语言中最常见的方法分派方式。
- invokeinterface 指令:用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找 出适合的方法进行调用。
- invokespecial 指令:用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和 父类方法。
- invokestatic 指令:用于调用类静态方法(static 方法)。
- invokedynamic 指令:用于在运行时动态解析出调用点限定符所引用的方法。并执行该方法。前面 四条调用指令的分派逻辑都固化在 Java 虚拟机内部，用户无法改变，而 invokedy namic 指令的分派逻辑 是由用户所设定的引导方法决定的。

方法调用指令与数据类型无关，而方法返回指令是根据返回值的类型区分的，包括 ireturn（当返回值是 boolean、byte、char、short 和 int 类型时使用）、lreturn、freturn、dreturn 和 areturn，另外还有一条 return 指令供声明为 void 的方法、实例初始化方法、类和接口的类初始化方法使用。

### 异常处理指令

在 Java 程序中显式抛出异常的操作(throw 语句)都由 athrow 指令来实现，除了用 throw 语句显式抛出异常的情况之外，《Java 虚拟机规范》还规定了许多运行时异常会在其他 Java 虚拟机指令检测到异常状态时自动抛出。例如整数运算中，当除数为零时，虚拟机会在 idiv 或 ldiv 指令中抛出 ArithmeticException 异常。

而在 Java 虚拟机中，处理异常（catch 语句）不是由字节码指令来实现的（很久之前曾经使用 jsr 和 ret 指令来实现，现在已经不用了），而是采用异常表来完成。

### 10. 同步指令

Java 虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor）来实现的。

方法级的同步是隐式的，无法通过字节码指令来控制，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池中的方法表结构中的 ACC_SYNCHRONIZED 访问标志得知一个方法是否被声明为同步方法。当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程有了管程，其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法边界之外时自动释放。

同步一段指令集序列通常是由 Java 语言中的 synchronized 语句块来表示，Java 虚拟机的指令集中有 monitorenter 和 monitorexit 两条指令来支持 synchronized 关键字的语义，正确实现 synchronized 关键字需要 javac 编译器与 Java 虚拟机两者共同协作支持。
