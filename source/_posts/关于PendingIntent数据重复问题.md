---
title: 关于PendingIntent数据重复问题
date: 2020-11-12 16:26:54
tags: 
  - android
categories:
  - 技术
description: 关于推送过程中遇到 PendingIntent 数据重复问题的解决。
---

之前在测试推送的过程中发现了一个很奇怪的问题，就是当收到多条推送时，点击推送获取的到的数据都是第一条推送的，经过查找之后，发现了该问题的原因。接下来一段就是来自官网的描述
> A PendingIntent itself is simply a reference to a token maintained by the system describing the original data used to retrieve it. This means that, even if its owning application's process is killed, the PendingIntent itself will remain usable from other processes that have been given it. If the creating application later re-retrieves the same kind of PendingIntent (same operation, same Intent action, data, categories, and components, and same flags), it will receive a PendingIntent representing the same token if that is still valid, and can thus call cancel() to remove it.
Because of this behavior, it is important to know when two Intents are considered to be the same for purposes of retrieving a PendingIntent. **A common mistake people make is to create multiple PendingIntent objects with Intents that only vary in their "extra" contents, expecting to get a different PendingIntent each time.** This does not happen. The parts of the Intent that are used for matching are the same ones defined by Intent#filterEquals(Intent). If you use two Intent objects that are equivalent as per Intent#filterEquals(Intent), then you will get the same PendingIntent for both of them.

加粗部分的意思是说，如果`PendingIntent`只有在`Intent`中的`extra`不同的话，那么`PendingIntent`只会拿到相同的，即第一次的那个，所以问题就出现在这里，官方给了两种这个情况的解决方案
> If you truly need multiple distinct PendingIntent objects active at the same time (such as to use as two notifications that are both shown at the same time), then you will need to ensure there is something that is different about them to associate them with different PendingIntents. This may be any of the Intent attributes considered by Intent#filterEquals(Intent), or different request code integers supplied to getActivity(Context, int, Intent, int), getActivities(Context, int, Intent[], int), getBroadcast(Context, int, Intent, int), or getService(Context, int, Intent, int).
If you only need one PendingIntent active at a time for any of the Intents you will use, then you can alternatively use the flags FLAG_CANCEL_CURRENT or FLAG_UPDATE_CURRENT to either cancel or modify whatever current PendingIntent is associated with the Intent you are supplying.

第一种方式就是给`Intent`设置不同的`filter`或者`request code`；第二种方式是如果不需要多个`PendingIntent`，那么可以在`flags`参数中传入`FLAG_CANCEL_CURRENT`或者`FLAG_UPDATE_CURRENT`将`PendingIntent`更新为最新的那个。因为我这边的情形是需要保留所有的不同的`PendingIntent`，所以采用了第一种方式，问题解决。
