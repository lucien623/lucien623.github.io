---
title: Flutter 更新失败问题
date: 2020-05-26 09:35:24
tags: 
  - Flutter
categories:
  - 技术
---
今天在执行`flutter upgrade`更新`flutter`时发现了一个更新失败问题：
```shell
Failed to retrieve the Dart SDK from: https://storage.googleapis.com/flutter_infra/flutter/6bc433c6b6b5b98dcf4cc11aff31cdee90849f32/dart-sdk-darwin-x64.zip
```

发生这个问题的主要原因就是这个地址被墙了，官方也给出了[解决方案](https://flutter.dev/community/china)，就是给更新地址设置 mirror site，具体的方法就是在 Bash shell 中做如下设置：
```shell
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
git clone -b dev https://github.com/flutter/flutter.git
export PATH="$PWD/flutter/bin:$PATH"
cd ./flutter
flutter doctor
```
除了这个 mirror site 之外，还可以用 Shanghai Jiaotong University Linux User Group 的 mirror site：
```java
FLUTTER_STORAGE_BASE_URL: https://mirrors.sjtug.sjtu.edu.cn/
PUB_HOSTED_URL: https://dart-pub.mirrors.sjtug.sjtu.edu.cn/
```
设置好之后就可以发会呆等待其 download 完成了。