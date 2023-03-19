---
title: Android 存储 API
date: 2021-12-14 19:56:02
tags: 
  - android
categories:
  - 技术
description: 关于 Android 存储 API 的简单笔记。
---

### internal storage 对应的存储目录

getCacheDir(): `/data/data/<application package>/cache`

getFilesDir(): `/data/data/<application package>/files`

### external storage 对应的存储目录

getExternalFilesDir(): `/sdcard/Android/data/<application package>/files`
 
getExternalCacheDir() : `/sdcard/Android/data/<application package>/cache`

一般来说手机外部存储的空间比较大，所以如果需要保存比较大的文件时，应当存储在外部存储中。internal storage 和 external storage 在 app 被删除后相应的文件都会被删除。

如果需要存储不跟随 app 删除的文件，可以通过获取外部存储路径的方式

```java
File sdCardFile = Environment.getExternalStorageDirectory();
File sdPic = new File(sdCardFile, "Pictures");
```

或者

```java
File sdDoc = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOCUMENTS);
```

前一种方式自己指定目录，后一种根据系统提供的参数获取目录。

在使用外部存储之前最好先加一个是否存在外部存储的判断

```java
//Environment.isExternalStorageEmulated() 判断设备的外存是否是用内存模拟
Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) || !Environment.isExternalStorageEmulated()
```

更多关于存储的开发指南可以去 [developer](https://developer.android.com/training/data-storage?hl=zh-cn) 网站参考。