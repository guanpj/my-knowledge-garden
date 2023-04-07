---
title: inline noinline crossinline
tags:
 - inline
 - noinline
 - crossinline
aliases: 
date created: 2023-04-06
date modified: 2023-04-07
---

# 普通函数

## 加入 inline 前

```kotlin
fun main() {  
	printName("guanpj")  
}  
  
fun printName(name: String) {  
	println("your name is: $name")  
}
```
反编译后：
```java
public static final void main() {  
	printName("guanpj");  
}

public static final void printName(@NotNull String name) {  
	String var1 = "your name is: " + name;  
	System.out.println(var1);  
}
```
输出结果：
	hello!
	your name is: guanpj
	bye!

## 加入 inline 后

```kotlin
public static final void main() {  
	printName("guanpj");  
}
  
inline fun printName(name: String) {  
	println("your name is: $name")  
}
```
反编译后：
```java
public static final void main() {  
	String name$iv = "guanpj";  
	String var2 = "your name is: " + name$iv;  
	System.out.println(var2);  
}

public static final void printName(@NotNull String name) {  
	String var2 = "your name is: " + name;  
	System.out.println(var2);  
}
```

# inline 对 lambda 参数的作用

## 加入 inline 前

```kotlin
fun main() {  
	printName("abc", { println("hello!") }, { println("bye!") })  
}  
  
fun printName(name: String, preAction: () -> Unit, postAction: () -> Unit) {  
	preAction()  
	println("your name is: $name")  
	postAction()  
}  
```
反编译结果：
```java
public static final void main() {  
	printName("guanpj", (Function0)null.INSTANCE, (Function0)null.INSTANCE);  
}  
  
public static final void printName(@NotNull String name, @NotNull Function0 preAction, 
								   @NotNull Function0 postAction) {  
	preAction.invoke();  
	String var3 = "your name is: " + name;  
	System.out.println(var3);  
	postAction.invoke();  
}
```
可以看到，main() 函数在调用 printName() 方法前，创建了一个 Function0 对象，对应类型为 `() -> Unit` 的 lambda 表达式。

因为 Java 并没有对函数类型的变量的原生支持，Kotlin 需要想办法来让这种自己新引入的概念在 JVM 中落地。而它想的办法是什么呢？就是用一个 JVM 对象来作为函数类型的变量的实际载体，让这个对象去执行实际的代码。也就是说，程序在每次调用 printName() 的时候都会创建一个对象来执行 lambda 表达式里的代码，虽然这个对象是用一下之后马上就被抛弃，但它确实被创建了。

这有什么坏处？其实一般情况下也没什么坏处，多创建个对象算什么？但是你想一下，如果这种函数被放在循环里执行，就会造成比较大的内存开支。

## 加入 inline 后

```kotlin
fun main() {  
	printName("guanpj", { println("hello!") }, { println("bye!") })  
}  
  
inline fun printName(name: String, preAction: () -> Unit, postAction: () -> Unit) {  
	preAction()  
	println("your name is: $name")  
	postAction()  
}
```
反编译结果：
```java
public static final void main() {  
	String name$iv = "guanpj";   
	String var3 = "hello!";  
	System.out.println(var3);  
	String var5 = "your name is: " + name$iv;  
	System.out.println(var5);  
	String var4 = "bye!";  
	System.out.println(var4);  
}  
  
public static final void printName(@NotNull String name, @NotNull Function0 preAction, 
								   @NotNull Function0 postAction) {  
	preAction.invoke();  
	String var4 = "your name is: " + name;  
	System.out.println(var4);  
	postAction.invoke();  
}
```
从句字节码中可以看到，经过这种优化，就避免了函数类型的参数所造成的临时对象的创建。也就不怕在循环或者界面刷新这样的高频场景里调用它们了。

## 注意事项

### public inline 方法不能私有属性

```kotlin
class Demo(private val title: String) {
	// 编译错误
    inline fun test(l: () -> Unit) {
        println("Title: $title") 
    }

    // 编译通过
    private inline fun test(l: () -> Unit) {
        println("Title: $title")
    }
}
```

### lambda 中的 return 造成意外返回

在普通函数中，是不允许 lambda 中直接 return 的。但是在内联函数中是允许的，如下所示：
```kotlin
fun main() {  
	printName("guanpj",  
		{  
			println("hello!")  
			return // 注意这里返回了 Unit
		},  
		{ println("bye!") }  
	)  
}  
  
inline fun printName(name: String, preAction: () -> Unit, postAction: () -> Unit) {  
	preAction()  
	println("your name is: $name")  
	postAction()  
}
```
执行结果：
hello!

反编译后：
```java
public static final void main() {  
	String name$iv = "guanpj";  
	int var2 = false;  
	String var3 = "hello!";  
	System.out.println(var3);  
}  
  
public static final void printName(@NotNull String name, @NotNull Function0 preAction, 
								   @NotNull Function0 postAction) {  
	preAction.invoke();  
	String var4 = "your name is: " + name;  
	System.out.println(var4);  
	postAction.invoke();  
}
```

#### 如何避免？

通过 @ 标签返回到指定位置：
```kotlin
fun main() {  
	printName("guanpj",  
		{  
			println("hello!")  
			return@printName  
		},  
		{ println("bye!") }  
	)  
}
```
输出结果：
hello!
your name is: guanpj
bye!

# noinline

noinline 的意思很直白：inline 是内联，而 noinline 就是不内联。不过它不是作用于函数的，而是作用于函数的参数：对于一个标记了 inline 的内联函数，你可以对它的任何一个或多个函数类型的参数添加 noinline 关键字。

## 加入 noinline 前

```kotlin
fun main() {  
	val postAction = printName("guanpj",  
		{ println("hello!") },  
		{ println("bye!") }  
	)  
	postAction()  
}  
  
inline fun printName(name: String, preAction: () -> Unit, postAction: () -> Unit): () -> 
Unit {  
	preAction()  
	println("your name is: $name")  
	return postAction // IDE 报错
}
```
编译器报错内容：
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/inline%26noinline%26crossinline/noinline.png)
把函数进行内联的时候，它内部的这些参数就不再是对象了，因为它们会被编译器拿到调用处去展开。

也就是说，main() 方法里面会将 printName() 方法进行了展开，同时会把 postAction 参数进行展开，从而获取不到 printName() 方法的返回值。

如果真的需要用这个对象，则需要加上 noinline 关键字。

## 加入 noinline 后

```kotlin
inline fun printName(name: String, preAction: () -> Unit, noinline postAction: 
					 () -> Unit): () -> Unit {  
	preAction()  
	println("your name is: $name")  
	return postAction  
}
```
反编译后：
```java
public static final void main() {  
	String name$iv = "guanpj";  
	Function0 postAction$iv = (Function0)null.INSTANCE;  
	String var5 = "hello!";  
	System.out.println(var5);  
	String var6 = "your name is: " + name$iv;  
	System.out.println(var6);  
	postAction$iv.invoke();  
}  
  
@NotNull  
public static final Function0 printName(@NotNull String name, @NotNull Function0 
										 preAction, @NotNull Function0 postAction) {  
	preAction.invoke();  
	String var4 = "your name is: " + name;  
	System.out.println(var4);  
	return postAction;  
}
```
可以看到，noline 加入后，preAction 没有被内联，并创建了一个 Function0 对象。

所以，noinline 的作用是什么？是用来局部地、指向性地关掉函数的内联优化的。既然是优化，为什么要关掉？因为这种优化会导致函数中的函数类型的参数无法被当做对象使用，也就是说，这种优化会对 Kotlin 的功能做出一定程度的收窄。而当你需要这个功能的时候，就要手动关闭优化了。这也是 inline 默认是关闭、需要手动开启的另一个原因：它会收窄 Kotlin 的功能。

那么，我们应该怎么判断什么时候用 noinline 呢？很简单，比 inline 还要简单：你不用判断，Android Studio 会告诉你的。当你在内联函数里对函数类型的参数使用了风骚操作，Android Studio 拒绝编译的时候，你再加上 noinline 就可以了。

# crossinline

## 突破间接调用

### 加入 crossinline 前

```kotlin
fun main() {  
	printName("guanpj",  
		{ println("hello!") },  
		{ println("bye!") }  
	)  
}  
  
inline fun printName(name: String, preAction: () -> Unit, postAction: () -> Unit) {  
	preAction()  
	println("your name is: $name")  
	runOnUiThread {  
		postAction() // IDE 报错
	}  
}  
  
fun runOnUiThread(runner: () -> Unit) {  
	println("Now in UI thread!")  
	runner()  
}
```
编译器报错内容：
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/inline%26noinline%26crossinline/crossinline.png)

### 加入 crossinline 后

```kotlin
inline fun printName(name: String, preAction: () -> Unit, crossinline postAction: () -> 
Unit) {  
	preAction()  
	println("your name is: $name")  
	runOnUiThread {  
		postAction()  
	}  
}
```
反编译：
```java
public static final void main() {  
	String name$iv = "guanpj";  
	String var3 = "hello!";  
	System.out.println(var3);  
	String var4 = "your name is: " + name$iv;  
	System.out.println(var4);  
	runOnUiThread((Function0)(new Test2Kt$main$$inlined$printName$1()));  
}  
  
public static final void printName(@NotNull String name, @NotNull Function0 
									preAction, @NotNull final Function0 postAction) {  
	preAction.invoke();  
	String var4 = "your name is: " + name;  
	System.out.println(var4);  
	runOnUiThread((Function0)(new Function0() {  
		public Object invoke() {  
			this.invoke();  
			return Unit.INSTANCE;  
		}  
	  
		public final void invoke() {  
			postAction.invoke();  
		}  
	}));  
}  
	  
public static final void runOnUiThread(@NotNull Function0 runner) {  
	String var1 = "Now in UI thread!";  
	System.out.println(var1);  
	runner.invoke();  
}
```
这下明白 crossinline 里的 `cross` 是什么意思了。
当然，如果把 runOnUiThread 也改成内联函数，也可以突破间接调用。

## 解决 lambda 中的 return 造成意外返回

另外，前面提到的 lambda 中的 return 造成意外返回的问题，也可以通过 crossinline 来解决：

### 加入 crossinline 前

```kotlin
fun main() {  
	printName("guanpj",  
		{  
			println("hello!")  
			return
		},  
		{ println("bye!") }  
	)  
}

inline fun printName(name: String, preAction: () -> Unit, postAction: () -> 
Unit) {  
	preAction()  
	println("your name is: $name")  
	postAction()  
}
```
输出结果：
hello!

### 加入 crossinline 后

```kotlin
fun main() {  
	printName("guanpj",  
		{  
			println("hello!")  
			return // IDE 报错
		},  
		{ println("bye!") }  
	)  
}

inline fun printName(name: String, crossinline preAction: () -> Unit, postAction: 
					 () -> Unit) {  
	preAction()  
	println("your name is: $name")  
	postAction()  
}
```
编译器报错内容：
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/inline%26noinline%26crossinline/crossinline1.png)

反编译：
```java
public static final void main() {  
	String name$iv = "guanpj";  
	String var3 = "hello!";  
	System.out.println(var3);  
	String var5 = "your name is: " + name$iv;  
	System.out.println(var5);  
	String var4 = "bye!";  
	System.out.println(var4);  
}  
  
public static final void printName(@NotNull String name, @NotNull Function0 preAction, 
								   @NotNull Function0 postAction) {  
	preAction.invoke();  
	String var4 = "your name is: " + name;  
	System.out.println(var4);  
	postAction.invoke();  
}
```
可以看到，添加了 crossinline 后反编译的代码其实跟没加的时候是一模一样的。因此 crossinline 关键字只是在语法上限制了被它修饰的 lambda 中使用 return 返回到外层函数。

# 总结

1.  inline 可以让你用内联——也就是函数内容直插到调用处的方式来优化代码结构，从而减少函数类型的对象的创建；
2.  noinline 是局部关掉这个优化，来摆脱 inline 带来的「不能把函数类型的参数当对象使用」的限制；
3.  crossinline 是局部加强这个优化，让内联函数里的函数类型的参数可以被当做对象使用，从而突破间接调用。