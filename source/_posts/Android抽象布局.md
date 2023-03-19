---
title: Android抽象布局
date: 2018-05-20 18:03:24
tags: 
  - android
categories:
  - 技术
description: 简单介绍 Android 中 &lt;include /&gt;、 &lt;merge /&gt; 和 &lt;ViewStub /&gt; 标签的使用。
---

### 1.`<include />` 标签
`<include />` 标签主要解决布局重用的问题，在我们大多数的 app 中的标题栏布局都是相似的，这个时候就可以用 `<include />` 标签重用布局文件了。使用方法如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
	xmlns:android="http://schemas.android.com/apk/res/android"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	android:orientation="vertical">

	<include layout="@layout/include_head"/>

</LinearLayout>
  ```

### 2.`<merge />` 标签
`<merge />` 标签的作用可以减少 UI 的层级，假如要在主布局 `LinearLayout` 中引入同样是 `LinearLayout` 的 `include` 布局，那么便可以将引入布局的最外层布局设置为 `<merge />` 标签，这样系统将会自动忽略 `<merge /> `节点，从而达到减少布局层次的目的。比如：
```xml
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="hello" />

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="world" />
    
</merge>
  ```

### 3.`<ViewStub />` 标签
`<ViewStub />` 标签的作用就是延迟加载布局资源，从而达到加快 UI 初始化速度目的，适用场景如进度条，加载失败布局等一些一开始不需要加载的布局。如：
```xml
<android.support.constraint.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.juhe.generaldemo.MainActivity">

    <ViewStub
        android:id="@+id/viewstub"
        android:layout="@layout/viewstub_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</android.support.constraint.ConstraintLayout>
  ```
想要加载布局时：
 ```java
  ViewStub viewStub = (ViewStub) findViewById(R.id.viewstub);
  viewStub.inflate();
 ```

