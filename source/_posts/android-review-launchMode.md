---
title: Activity启动模式
date: 2017-07-10 11:13:35
tags: 
  - android 
categories:
  - 技术
description: 简单介绍下在 Android 中的四种 Activity 启动模式。
---


### launchMode

- standard
  默认的启动方式，每次都会创建一个活动的实例。  
  task 1: A-B-C-D  
  add D  
  task 1: A-B-C-D-D

- singleTop
  当有活动实例存在于栈顶时，那么将不会生成一个新的活动实例，此时会调用栈顶实例的 onNewIntent() 方法。  
  task 1: A-B-C-D  
  add D  
  task 1: A-B-C-D

- singleTask
  当栈中存在该活动的实例时，那么启动活动时，栈中位于该活动实例上方的活动都将会被移除栈，并调用该活动的 onNewIntent();  
  task 1: A-B-C-D  
  add C  
  task 1: A-B-C

- singleInstance
  启动活动时会创建一个新的栈  
  task 1: A-B-C-D  
  add D  
  task 1: A-B-C-D  
  task 2: D  
  场景：  
  调用相机

