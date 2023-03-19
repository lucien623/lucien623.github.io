---
title: Java反射机制
date: 2018-08-11 21:53:45
tags: 
  - Java 
categories:
  - 技术
description: 对 Java 反射机制的简要总结。 
---

### 什么是反射？

> Reflection is a feature in the Java programming language. It allows an executing Java program to examine or "introspect" upon itself, and manipulate internal properties of the program. For example, it's possible for a Java class to obtain the names of all its members and display them. --- [from oracle](https://www.oracle.com/technetwork/articles/java/javareflection-1536171.html)

大致意思是反射是 Java 语言的特性之一，它允许一个运行中的 Java 程序去获取自身的信息并操作它们。

首先我们定义一个 Student 类：
``` java
class Student {
	public int age;
	protected String name;
	public String gender;
	
	public Student() {
		
	}
	
	public Student(int age, String name, String gender) {
		this.age = age;
		this.name = name;
		this.gender = gender;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getGender() {
		return gender;
	}

	public void setGender(String gender) {
		this.gender = gender;
	}
}
```

### 获取 Class 对象

获取 Class 对象有如下三种方式：
``` java
Class clsA = Class.forName("com.djp.demo.Student");
Class clsB = Student.class;
Class clsC = new Student().getClass();
```

### 获取方法

``` java
private static void getMethods(Class cls) {
	Method[] methods = cls.getDeclaredMethods();
	for (Method method : methods) {
		System.out.println(method.toString());
	}
}
```
通过`getDeclaredMethods()`方法可以返回类或者接口的所有方法，包括 public, protected, defalut, and private，但无法获取到继承的方法。打印信息如下：
``` text
public java.lang.String com.djp.demo.Student.getName()
public void com.djp.demo.Student.setName(java.lang.String)
public int com.djp.demo.Student.getAge()
public java.lang.String com.djp.demo.Student.getGender()
public void com.djp.demo.Student.setGender(java.lang.String)
public void com.djp.demo.Student.setAge(int)
```
另一个`getMethods()`方法也可以获取到类和接口的所有方法，与上面方法不同的是调用这个方法只会返回定义为 public 的 Method 对象，包括继承的方法。

### 创建实例

有两种方法可以创建实例，第一种是直接通过 class 对象的 newInstance 创建：
``` java
Class clsA = Class.forName("com.djp.demo.Student");
clsA.newInstance();
```
第二种通过构造器创建：
``` java
Constructor constructor = cls.getConstructor(int.class, String.class, String.class);
Student student = (Student) constructor.newInstance(18, "lucien", "man");
```

### 通过 name 调用方法

首先根据 name 获取到 Method 对象，然后调用 invoke 方法传入对应参数就可以调用方法了，代码如下：
``` java
Method method = cls.getMethod("getAge");
method.invoke(new Student(22,"lucien",  "man"));
```

### 获取成员变量
通过调用 `getDeclaredFields()` 或者 `getFields()`，方法区别和获取方法类似。
