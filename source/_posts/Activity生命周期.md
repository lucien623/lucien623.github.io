---
title: Activity生命周期
date: 2017-07-10 11:37:39
tags: 
  - android
categories:
  - 技术
---

Activity 是 Android 四大组件之一，它主要给用户提供了一个交互页面，用户通过它与app进行交互，以执行各种操作。 每个 Activity 都会获得一个用于绘制其用户界面的窗口。窗口通常会充满屏幕，但也可小于屏幕并浮动在其他窗口之上（将 Activity 的 theme 设置为 Dialog）。Activity 生命周期中主要的七个回调方法介绍如下表：


| 方法        | 说明   |
| --------   | -----  |
| onCreate()     | Activity 创建的时候调用,可以在方法里做一些初始化的操作，比如初始化试图，获取从上一个页面传递过来的数据，此时处于不可见状态。 |
| onRestart()        |   在 Activity 已停止并即将再次启动前调用。   |
| onStart()        |   Activity 处于可见状态，但不能操作，因为并未获取到焦点。   |
| onResume()        |    Activity 处于可见状态，并且能够获取到焦点，在 Activity 恢复时可以在此方法中刷新数据。    |
| onPause()        |    当系统即将开始继续另一个 Activity 时调用。当此方法执行完才会启动下一个 Activity，此方法中可以做一些停止动画，释放一些资源的操作，但不适合一些比较耗时的操作，因为这样会导致跳转的时候卡顿。    |
| onStop()        |    在 Activity 对用户不再可见时调用。如果 Activity 恢复交互，那么接下来会调用 onRestart() 方法，如果 Activity 销毁，那么将会调用 onDestory() 方法。   |
| onDestory()        |    在 Activity 被销毁前调用。    |

![activity-life](https://developer.android.com/images/activity_lifecycle.png)

