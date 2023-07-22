---
title: Java异常处理中throw和throws的区别和用法
date: 2018-07-10 17:41:56
categories:
- Java
tags:
- Java
---

> 前一篇文章已经详细介绍了 Java 的异常处理机制，在这里做一些补充，探讨异常处理关键字 throw 和 throws 的区别和用法。转载请说明出处。

<!--more-->

## 抛出异常的三种形式
Java 抛出异常有三种形式：

* 系统自动抛出异常
* 方法名通过 throws 抛出异常
* 方法体通过 throw 抛出异常

### 系统自动抛出异常
当程序语句出现一些逻辑错误、主义错误或类型转换错误时，系统会自动抛出异常。发生异常后线程停止运行，后面的语句执行不到，会在包含它的所有 try 块中（可能在上层调用函数中）从里向外寻找含有与其匹配的 catch 子句的 try 块。如：

```
public static void main(String[] args) {
	int a = 5, b =0;
	System.out.println(5/b);
	//function();
}
```

系统会自动抛出ArithmeticException异常：

```
Exception in thread "main" java.lang.ArithmeticException: / by zero

at test.ExceptionTest.main(ExceptionTest.java:62)
```

再如

```
public static void main(String[] args) {
	String s = "abc";
	System.out.println(Double.parseDouble(s));
	//function();
}
```

系统会自动抛出NumberFormatException异常：

```
Exception in thread "main" java.lang.NumberFormatException: For input string: "abc"

at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1224)

at java.lang.Double.parseDouble(Double.java:510)

at test.ExceptionTest.main(ExceptionTest.java:62)
```

### throw 抛出异常
throw 出现在方法体中，用于程序员主动抛出某种特定类型的异常。通过 throw 抛出的异常和系统自动抛出的异常有相同的效果。

```

public static void main(String[] args) {
	String s = "abc";
	if(s.equals("abc")) {
		throw new NumberFormatException();
	} else {
		System.out.println(s);
	}
	//function();
}
```

会抛出异常：

```
Exception in thread "main" java.lang.NumberFormatException

at test.ExceptionTest.main(ExceptionTest.java:67)
```

### throws 抛出异常
throws 出现方法头中，用来声明该方法可能抛出的异常类型，然后把异常交给调用它的上级程序来处理。

对于 Error 和 RuntimeException 类型的及其子类的异常，不需要在方法头通过 throws 明确声明，因为这是不期待出现的异常。对于其他类型的异常 Java 编译器强制要求在方法头 通过 throws 明确声明，否则编译不通过。

```
public static void function() throws NumberFormatException{
	String s = "abc";
	System.out.println(Double.parseDouble(s));
}
	
public static void main(String[] args) {
	try {
		function();
	} catch (NumberFormatException e) {
		System.err.println("非数据类型不能转换。");
		//e.printStackTrace();
	}
}
```

输出结果如下：

```
非数据类型不能转换。
```

## throw 和 throws 的区别和用法

### 区别
* throws 用于方法头，throw 用于方法内部
* throws 后跟异常类型，表示可能会出现这些类型的异常，但是不一定抛出；throw 后跟异常对象，如果执行到 throw 就一定会抛出这种类型的异常。
* throw 后只能跟一个异常对象，throws 后可以一次声明多种异常类型。

### 使用场合
Java 将派生于 Error 类和 RuntimeException 类的异常称为 **非检查异常** (unchecked exception)，将其他异常称为 **检查异常** (checked exception)。

一般来说，系统在编译阶段检查的是检查异常和 throws 声明的异常，在运行阶段自动抛出的是非检查异常。

对于检查异常，如果通过 throw 主动抛出，或者调用了可能会产生异常的方法但是又没有进行处理，必须在方法头上用 throws 显式声明该异常类型，否则编译器会报错。如下面 IOException 为检查异常

```
//通过throw主动抛出
public void throwEx1() throws IOException { //正常编译
    throw new IOException("."); 
}

public void throwEx2() { //编译报错 unhandled exception java.io.ioexeption
    throw new IOException(".");
}

//调用可能会产生异常的方法但是没有进行处理
public void throwEx3() throws IOException{ //正常编译
    FileInputStream input = new FileInputStream("xxx");
}

public void throwEx4() { //编译报错 unhandled exception java.io.ioexeption
    FileInputStream input = new FileInputStream("xxx");
}
```

对于非检查异常，如果通过 throw 主动抛出，可自行选择是否在方法头用 throws 显式声明，不声明编译也不会报错。如下面 SQLException 为非检查异常

```
public void throwEx5() throws SQLException{ //正常编译
    throw new IllegalStateException(".");
}

public void throwEx6() { //正常编译
    throw new IllegalStateException(".");
}
```

如果调用方法名包含 throws 异常类型的方法，必须*在方法内通过 `try...catch` 捕获，或者继续在方法名上用 throws 声明继续往上抛*。例如调用上面的方法

```
public void throwEx7() {
    throwEx1(); //编译报错 unhandled exception java.io.ioexeption
    throwEx5(); //编译报错 unhandled exception java.sql.sqlexeption
    throwEx6(); //正常编译
}

public void throwEx8() throws IOException, SQLException{
    throwEx1(); //正常编译
    throwEx5(); //正常编译
    throwEx6(); //正常编译
}
```

### 编程总结
1. 在写程序时，对可能会出现异常的部分通常要用 `try{...}catch{...}` 去捕捉它并对它进行处理
2. 用 `try{...}catch{...}` 捕捉了异常之后一定要对在 `catch{...}` 中对其进行处理，那怕是最简单的一句输出语句，或栈输入 `e.printStackTrace()`
3. 如果是捕捉 IO 输入输出流中的异常，一定要在 `try{...}catch{...}` 后加 `finally{...}` 把输入输出流关闭；
4. 如果在函数体内用 throw 抛出了某种异常，最好要在函数名中加 throws 抛异常声明，然后交给调用它的上层函数进行处理。

## 参考
[再探java基础——throw与throws](https://blog.csdn.net/luoweifu/article/details/10721543)  
[百度问答](https://zhidao.baidu.com/question/519014936.html?si=2&qbpn=1_2&tx=&wtp=wk&word=Java%EF%BC%9Athrow%E5%92%8Cthrows%E6%9C%89%E5%BF%85%E8%A6%81%E5%90%8C%E6%97%B6%E4%BD%BF%E7%94%A8%E5%90%97%EF%BC%9F&fr=solved&from=qb&ssid=&uid=bd_1492636518_230&pu=sz%40224_240%2Cos%40&step=2&bd_page_type=1&init=middle)