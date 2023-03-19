---
title: Android事件分发
date: 2017-07-11 17:06:00
tags: 
  - android
categories:
  - 技术
---

Android 中和事件分发相关的主要有三个方法，分别是 dispatchTouchEvent(...)、onInterceptTouchEvent(...) 和 onTouchEvent(...)，主要作用是分发事件、是否拦截事件以及处理事件，这些方法的返回值决定了 Touch 事件的传递方向，方法的包涵情况具体如下表所示：

| | dispatchTouchEvent(...)|onInterceptTouchEvent(...)|onTouchEvent(...)|
| --------   | :-----: | :-----:  | :-----: |
| Activity   |   yes   |    no    |   yes   |
| ViewGroup  |   yes   |    yes   |   yes   |
| view       |   yes   |    no    |   yes   |

以下为事件分发流程图（针对于 ACTION_DOWN 事件，可点击查看大图）

<!-- touchevent.png -->
![](http://wx4.sinaimg.cn/large/9302210cly1fygi1siu3qj21ex0t876w.jpg)


如图所示的事件传递流向已十分清晰，demo 也十分易写，只需自行重写这些方法，打印日志便可验证。

对于 ACTION_MOVE 和 ACTION_UP 事件的传递则略有不同，它们的传递和 ACTION_DOWN 事件传递的终点相关，以下举例。

#### 1）在 View 的 onTouchEvent 消费事件，即 return true。

日志：

<!-- touchevent_2_log -->

![](http://wx3.sinaimg.cn/large/9302210cly1fygi14xf2mj21ww08in3n.jpg)

事件传递的流向图：

<!-- touchevent_2.png -->

![](http://wx1.sinaimg.cn/large/9302210cly1fygi18l5huj21gd0u0gt7.jpg)

#### 2）在 ViewGroup 的 onTouchEvent(...) 消费事件，即 return true。

日志：

<!-- touchevent_1_log -->

![](http://wx4.sinaimg.cn/large/9302210cly1fygi0wgpg3j21ww07mq8v.jpg)

事件传递的流向图：

<!-- touchevent_1 -->

![](http://wx2.sinaimg.cn/large/9302210cly1fygi10hcufj21gb0u0dn6.jpg)

#### 3）在 ViewGroup 的 dispatchTouchEvent(...) 方法中 return false。

日志：
 
<!-- touchevent_3_log -->

![](http://wx4.sinaimg.cn/large/9302210cly1fygi1c1kczj21ww044jup.jpg)

事件传递流向图：

<!-- touchevent_3 -->

![](http://wx1.sinaimg.cn/large/9302210cly1fygi1oxebij21gh0u0tfs.jpg)







