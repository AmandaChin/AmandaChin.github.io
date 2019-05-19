---
layout:     post
title:      《第一行代码》学习笔记
subtitle:   Activity基础用法
date:       2019-05-15
author:     Ann
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Android初学
    - 学习笔记
---

> 小萌新准备学安卓，最近在读《第一行代码》，感觉作者写的好朗朗上口，清晰明了啊～连玩带学的看了两章，记个笔记以后翻出来看看改改～嘻嘻

### 基本知识

Activity是安卓四大组件之一，它是展现在用户面前的组件，类似于jsp。

Android讲究逻辑视图分离，每个活动对应一个视图，视图写在app->src->main->res->Layout中

逻辑写在src->java->包名中

**每个Activity都要在AndroidManifest.xml中注册**

### 生命周期
Android用任务（Task）管理活动， 每个活动都会被放在返回栈中， 当前活动即为棧顶元素。

每个活动有四种状态：

- 运行状态（棧顶）
- 暂停状态
- 停止状态
- 销毁状态（出棧）
 
对应于八种生存状态：  
![cmd-markdown-logo](/img/activity-life.png)

其中onDestroy() 和 onCreate() 对应  
onPause 和 onResume对应  
onStop后会先Restart再start

### 活动四种启动模式
**在AndroidManifest.xml中为activity标签指定android:launchMode选择启动模式**
```xml
 <activity android:name=".DialogActivity"
            android:launchMode="singleTop">
</activity>
```
- standard
    - 每次启动activity都在棧顶插入
- singleTop
    - 待打开的activity在棧顶时不会重复插入，但是当前棧顶元素不是该活动时，再次调用会重复插入
- singleTask
    - 只要棧中有该activity就不会被重复插入
- singleInstance
    - 被指定为singleInstance的活动，当被调用时会放到一个新的棧中。
    - 一般每个应用程序都有一个自己的返回棧，通过这个方式可以实现多个程序共享同一个活动。
