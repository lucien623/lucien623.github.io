---
title: ButterKnife原理探析
date: 2019-10-19 19:03:09
tags: 
  - android
categories:
  - 技术
description: 对 ButterKnife 原理的简要分析。
---

`ButterKnife`是`Android`开发中常用的`View`注入框架，主要作用就是减少`findViewById`代码，使其更简洁。在分析这个框架之前我们先要了解一下`Annotation` 注解相关的知识。

注解是`Java`语言 5.0 版本开始支持加入源代码的特殊语法元数据。`Java`语言中的类、方法、变量、参数和包等都可以被标注。主要有三个作用，第一种作用是标记用来告诉编译器一些信息；第二种是编译时处理动态生成代码；第三种是运行时动态处理得到注解信息。注解主要分为三类：

标准注解：是`Java`自带的一些注解，比如`@Override`、`@Deprecated`

元注解：指用来定义自定义注解的注解，`@Retention`表示注解的保留时间，`SOURCE`表示源码时保留编译后会被抛弃，`CLASS`表示编译时保留运行时会被丢弃，`RUNTIME`表示运⾏时保留。`@Target`用来表示注解可以修饰哪些元素，比如`TYPE`(类、接口)，`FIELD`（字段），`METHOD`（方法），`CONSTRUCTOR`（构造方法）, `PARAMETER`（参数申明）等。`@Inherited`表示是否可以被继承，`@Documented`是否会保存到⽂档。

自定义注解：通过元注解定义的注解，我们来看以下`ButterKnife`中的一个自定义注解
```java
@Retention(CLASS) 
@Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```
我们可以看到这个注解会在编译时被保留`@Retention(CLASS)`，目标是对于字段`@Target(FIELD)`，注解的类型是`int`类型的`id`资源。

好了，注解大概就分析这些，接下来来看看`ButterKnife`是如何实现的。

首先`ButterKnife`不仅可以注入`View`，还可以绑定资源，点击事件等等，使用方式就不一一讲解了，ok，那么我们以绑定`view`为例，比如我们在`LoanFragment`里添加了如下代码
```java
@BindView(R.id.text_title) TextView mTextTitle;
```
然后我们在初始化的时候调用绑定`ButterKnife.bind(this, mFragmentView);`方法
```java
public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
    TAG = this.getClass().getSimpleName();
    if (mFragmentView == null) {
        mFragmentView = inflater.inflate(getLayoutId(), container, false);
        unbinder = ButterKnife.bind(this, mFragmentView);
        initView(mFragmentView);
        initData();
    }
    return mFragmentView;
}
```
接着看`bind`方法里的逻辑
```java
@NonNull @UiThread
public static Unbinder bind(@NonNull Object target, @NonNull View source) {
  return createBinding(target, source);
}

private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
  Class<?> targetClass = target.getClass();
  if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

  if (constructor == null) {
    return Unbinder.EMPTY;
  }

  //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
  try {
    return constructor.newInstance(target, source);
  } catch (IllegalAccessException e) {
    throw new RuntimeException("Unable to invoke " + constructor, e);
  } catch (InstantiationException e) {
    throw new RuntimeException("Unable to invoke " + constructor, e);
  } catch (InvocationTargetException e) {
    Throwable cause = e.getCause();
    if (cause instanceof RuntimeException) {
      throw (RuntimeException) cause;
    }
    if (cause instanceof Error) {
      throw (Error) cause;
    }
    throw new RuntimeException("Unable to create binding instance.", cause);
  }
}
```
第一步获取到`LoanFragment`的`Class`实例，接着调用`findBindingConstructorForClass`
```java
@Nullable @CheckResult @UiThread
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
  Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
  if (bindingCtor != null) {
    if (debug) Log.d(TAG, "HIT: Cached in binding map.");
    return bindingCtor;
  }
  String clsName = cls.getName();
  if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
    if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
    return null;
  }
  try {
    Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
    //noinspection unchecked
    bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
    if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
  } catch (ClassNotFoundException e) {
    if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
    bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
  } catch (NoSuchMethodException e) {
    throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
  }
  BINDINGS.put(cls, bindingCtor);
  return bindingCtor;
}
```
这一步是为了获取相对应类的构造器，`BINDINGS`是一个保存着`Class`实例和构造器相映射的`map`，进入这个方法会先去判断之前是否已经获取过该`Class`实例的构造器，如果有则直接`return`，否则获取到该类的名称，通过该`Class`实例的类加载器把`clsName + "_ViewBinding"`加载，再通过这个`Class`实例获取到相应的构造器返回并放入`BINDINGS`中，`clsName + "_ViewBinding"`到底是什么呢，其实在编译之后我们可以在`build/generated/source/apt`路径下可以找到`LoanFragment_ViewBinding`这个类，正好是这个拼接字符串的格式，拿到构造器之后会通过`return constructor.newInstance(target, source);`构建出相应实例，那么继续看这个构造方法里执行了什么
```java
@UiThread
public LoanFragment_ViewBinding(LoanFragment target, View source) {
  this.target = target;

  target.mTextTitle = Utils.findRequiredViewAsType(source, R.id.text_title, "field 'mTextTitle'", TextView.class);
  //...此处省略部分代码
}

public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
    Class<T> cls) {
  View view = findRequiredView(source, id, who);
  return castView(view, id, who, cls);
}

public static View findRequiredView(View source, @IdRes int id, String who) {
  View view = source.findViewById(id);
  if (view != null) {
    return view;
  }
  String name = getResourceEntryName(source, id);
  //...此处省略部分代码
}
```
很明显了，如果是注入`view`的话最终还是调用`source.findViewById(id)`方式获取到的，`target.mTextTitle`即我们在`LoanFragment`中定义的`mTextTitle`，`target`是我们调用`bind`方法是传进来的`LoanFragment`实例。当`Activity`调用`bind`方法时
```java
public AboutUsActivity_ViewBinding(AboutUsActivity target) {
  this(target, target.getWindow().getDecorView());
}
```
这里可以看到会获取当前`Activity`的`DecorView`，即最顶层布局作为`source`调用`findViewById`的。`bind`的整体流程大致就是如此了，至于那些编译时的注入类主要依靠的是 apt 工具生成的，就暂时不展开了。


参考文章：
http://trinea.github.io/download/pdf/android/java-annotation.pdf