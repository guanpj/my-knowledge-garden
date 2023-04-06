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

没有 inline
```kotlin
fun main() {  
	val res = sum(1, 2)  
	println("Result is: $res")
}  
  
fun sum(a: Int, b: Int): Int {  
	return a + b  
}
```
反编译后：
```java
public static final void main() {  
	int res = sum(1, 2);  
	String var1 = "Result is: " + res;  
	System.out.println(var1);
}

public static final int sum(int a, int b) {  
	return a + b;  
}
```
加入 inline：
```kotlin
fun main() {  
	val res = sum(1, 2)  
	println("Result is: $res")
}  
  
inline fun sum(a: Int, b: Int): Int {  
	return a + b  
}
```
反编译后：
```java
public static final void main() {  
	byte a$iv = 1;  
	int b$iv = 2;  
	int $i$f$sum = false;  
	int res = a$iv + b$iv;  
	String var4 = "Result is: " + res;  
	System.out.println(var4);  
}

public static final int sum(int a, int b) {  
	int $i$f$sum = 0;  
	return a + b;  
}
```

# inline 的作用

内联之前：
```kotlin
fun main() {
	println("Start")  
	sum(1, 2) {
		println("Result is: $it")  
	}  
	println("Done")  
}

fun sum(a: Int, b: Int, printResult: (result: Int) -> Unit): Int {  
	val r = a + b  
	printResult(r)  
	return r  
}  
```
反编译结果：
```java
public static final void main() {  
	String var0 = "Start";  
	System.out.println(var0);  
	sum(1, 2, (Function1)null.INSTANCE);  
	var0 = "Done";  
	System.out.println(var0);  
}

public static final int sum(int a, int b, @NotNull Function1 printResult) {  
	Intrinsics.checkNotNullParameter(printResult, "printResult");  
	int r = a + b;  
	printResult.invoke(r);  
	return r;  
}
```
内联之后：
```kotlin
fun main() {  
	println("Start")  
	sum(1, 2) {  
		println("Result is: $it")  
	}  
	println("Done")  
}

inline fun sum(a: Int, b: Int, printResult: (result: Int) -> Unit): Int {  
	val r = a + b  
	printResult(r)  
	return r  
}
```
反编译结果：
```java
public static final void main() {  
	String var0 = "Start";  
	System.out.println(var0);  
	byte a$iv = 1;  
	int b$iv = 2;  
	int $i$f$sum = false;  
	int r$iv = a$iv + b$iv;  
	int var5 = false;  
	String var6 = "Result is: " + r$iv;  
	System.out.println(var6);  
	var0 = "Done";  
	System.out.println(var0);  
}

public static final int sum(int a, int b, @NotNull Function1 printResult) {  
	int $i$f$sum = 0;  
	Intrinsics.checkNotNullParameter(printResult, "printResult");  
	int r = a + b;  
	printResult.invoke(r);  
	return r;  
}
```

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

### lamda 中的 return 造成意外返回

```kotlin
fun main() {  
	println("Start")  
	sum(1, 2) {  
		println("Result is: $it")  
		return //注意这里返回了 Unit
	}  
	println("Done")  
}  
  
inline fun sum(a: Int, b: Int, printResult: (result: Int) -> Unit): Int {  
	val r = a + b  
	printResult(r)  
	return r  
}
```
执行结果：
Start
Result is: 3

反编译后：
```java
public static final void main() {  
	String var0 = "Start";  
	System.out.println(var0);  
	byte a$iv = 1;  
	int b$iv = 2;  
	int $i$f$sum = false;  
	int r$iv = a$iv + b$iv;  
	int var5 = false;  
	String var6 = "Result is: " + r$iv;  
	System.out.println(var6);  
}

public static final int sum(int a, int b, @NotNull Function1 printResult) {  
	int $i$f$sum = 0;  
	Intrinsics.checkNotNullParameter(printResult, "printResult");  
	int r = a + b;  
	printResult.invoke(r);  
	return r;  
}
```
如何避免？
返回到指定位置：
```kotlin
fun main() {  
	println("Start")  
	sum(1, 2) {  
		println("Result is: $it")  
		return@sum  
	}  
	println("Done")  
}
```
输出结果：
Start
Result is: 3
Done
# crossinline
```kotlin
fun main() {  
	println("Start")  
	sum(1, 2) {  
		println("Result is: $it")  
		return // 编译报错，不允许返回
	}  
	println("Done")  
}  
  
inline fun sum(a: Int, b: Int, crossinline printResult: (result: Int) -> Unit): Int {  
	val r = a + b  
	printResult(r)  
	return r  
}
```
# noinline
```kotlin
fun main() {  
	println("Start")  
	sum(1, 2,  
		{ println("Result is: $it") },  
		{ println("Deal result: $it") }  
	)  
	println("Done")  
}  
  
inline fun sum(a: Int, b: Int, printResult: (result: Int) -> Unit, noinline 
			   dealResult: (result: Int) -> Unit): Int {  
	val r = a + b  
	printResult(r)  
	dealResult(r)  
	return r  
}
```
反编译后：
```java
public static final void main() {  
	String var0 = "Start";  
	System.out.println(var0);  
	byte a$iv = 1;  
	byte b$iv = 2;  
	Function1 dealResult$iv = (Function1)null.INSTANCE;  
	int $i$f$sum = false;  
	int r$iv = a$iv + b$iv;  
	int var6 = false;  
	String var7 = "Result is: " + r$iv;  
	System.out.println(var7);  
	dealResult$iv.invoke(r$iv);  
	var0 = "Done";  
	System.out.println(var0);  
}

public static final int sum(int a, int b, @NotNull Function1 printResult, 
							@NotNull Function1 dealResult) {  
	int $i$f$sum = 0;  
	Intrinsics.checkNotNullParameter(printResult, "printResult");  
	Intrinsics.checkNotNullParameter(dealResult, "dealResult");  
	int r = a + b;  
	printResult.invoke(r);  
	dealResult.invoke(r);  
	return r;  
}
```