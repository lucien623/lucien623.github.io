---
title: 记关于Fragment-java.lang.NoSuchMethodException的一个报错
date: 2022-02-15 13:55:48
tags: 
  - android
categories:
  - 技术
description: rt.
---
春节回来之后瞄了眼 bugly，发现了一个发生频次高影响用户广的 bug:

`java.lang.RuntimeException:Unable to start activity ComponentInfo{.ui.activity.HomeActivity}: androidx.fragment.app.Fragment$InstantiationException: Unable to instantiate fragment .ui.fragment.q: could not find Fragment constructor`

搜索了问题之后发现原因很简单，同事在创建的`Fragment`的时候用了带参数的构造方法，而没有添加无参构造，从而导致了这个 runtime exception，来看一下官方文档的解释：

`All subclasses of Fragment must include a public no-argument constructor. The framework will often re-instantiate a fragment class when needed, in particular during state restore, and needs to be able to find this constructor to instantiate it. If the no-argument constructor is not available, a runtime exception will occur in some cases during state restore`

从这段说明中可以看出所有的`Fragment`子类都必须包含无参的构造方法，系统在根据需要重新实例化`Fragment`子类时会默认去调用无参构造方法，如果没有找到这个无参方法，则会导致运行时错误。