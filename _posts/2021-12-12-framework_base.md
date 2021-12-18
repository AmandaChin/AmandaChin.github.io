---
layout:     post
title:      FrameWork基础
subtitle:   Binder & AIDL
date:       2021-12-12
author:     Ann
header-img: img/bg_eden.jpeg
catalog: true
tags:
    - framework
    - Android
    - 插件化
---

> 说到FrameWork底层，离不开binder通信。之前对于binder总是有种云里雾里的感觉，拜读了包老师的《插件化开发指南》，对于binder的通信原理有了一定的理解，希望后续能够深入代码，进一步补充细节:)

### Binder原理
- Client & Server  
binder通信是基于IPC（跨进程通信）的，故有Client进程和Server进程的概念。他们是相对的，发送消息方为Client进程，接收消息方为Server进程。

- Binder组成  
![](https://upload-images.jianshu.io/upload_images/2116778-ecd063b4e693dccb.png?imageMogr2/auto-orient/strip|imageView2/2/w/798/format/webp)

进程间通信依赖于ServiceManager。对于Server端，需要向ServiceManager注册具体服务。对于Client端，基于binder驱动访问时，像ServiceManger查询是否相应的服务已注册。

> 对于相关的两个需要通讯的进程，它们通过调用`libutil.so`库实现通讯，而真正通讯的机制，是内核空间中的一块`共享内存`。


- Binder通信过程
    - Server在SM（ServiceManager）中注册服务。
    - Client中调用Server方法A()，请求SM进行查询，SM确定Server是否被注册，并返回一个`代理类`。
    - 代理类调用真正的方法A，并把结果返回给Client。

- Binder限制传输数据大小为1M，只适用于轻量数据IPC，大型数据可考虑ContentProvider。

### AIDL原理
> AIDL是binder的延伸，Android中许多系统方法都是使用aidl进行通信的。如剪切板，四大组件通信等。

重要的几个类：
- IBinder
- IInterface
- Stub
- Proxy

利用aidl工具生成一个IxxxxService.java文件，它的uml图如下。  
!()[https://img-blog.csdn.net/20130703113807968]

即，每个生成的aidl类中包含三个主要的类。
- IInterface （接口类，重要方法：*** 通过aidl通信的方法接口）
- Stub (重要的类！他实现了binder类)
- Proxy (代理，通过Parcelable序列化数据，并调用IBinder的transact方法，和具体的server通信)

Stub都做了哪些工作？  
Stub继承了binder并实现了aidl接口。主要有以下接口：
- asBinder 返回this 对应的binder
- asInterface(IBinder obj) Client端调用，用来判断当前传参`obj`是否与自己在同一个进程中，若在，则无需跨进程通信；否则，将`obj`包装为代理类`Proxy`，并返回。
- onTransact() Server端调用，接收代理类proxy发送过来的数据（序列化传递）并执行相应的操作。

### 四大组件通信
#### 启动Activity流程
- Launcher/Activity 通知 AMS 需要启动***Activity。
- AMS记录信息并进行校验，通知 Launcher/Activity 可以休息。
- Launcher/Activity 页面进入Paused状态， 通知 AMS 已经休息。
- （Launcher情况，AMS判断是否需要开启新的进程，若需要，进行相关初始化操作，创建新进程，并启动ActivityThread中的main方法）
- AMS 通知待启动App 启动信息
- 具体的App启动页面，创建context与Activity关联，并调用Activity的onCreate/onNewIntent方法。

可以看出来他们是双向通信的，首次发送请求的Activity为Client端，接收并响应消息的AMS为Server端。

- Activity -> Server  
类比aidl流程图，有如下几个类：  
AMS（ActivityManagerService），AMN（ActivityManagerNative）,AMP(ActivityManagerProxy)
    - AMP 即为代理类，负责处理消息，并通过binder调用到真实实现。
    - AMS 即为Server端的实现（类比于Stub的onTransact)
    - AMN 在Client端，类比Stub的asInterface，用来判断binder是否需要跨进程通信，通过AMN.getDefault拿到具体的server端binder。（`这里也是插件化hook的关键点`)

- Server -> Activity  
由于Activity先调用的AMS，在AMS端有一个`ActivityRecord`类去存储调用方的相关信息（进程id等），ActivityRecord里持有`ApplicationThreadProxy`，于是，类似于Client端通过proxy调用真实server一样，AMS 通过 ApplicationThreadProxy 调用 Activity端的ApplicationThread

##### 一些重要类
- AMN、AMP
- ApplicationThread
    - Activity进程加载首要初始化类，这里面包含了main方法，他主要的工作有：初始化looper和messageQueue对象、创建ActivityThread（UI县城）对象、建立binder通信。
- H
    - 位于ActivityThread内，是个handeler，四大组件的创建message都经由他来处理。
- Instrumentation
    - 仪表盘，创建Activity的具体实现类，它里面只有对Activity的处理逻辑。

- Context
    - Activity、Service、Application都继承与context，Activity继承了包装类`contextThemeWrapper`
    - ContextImpl为具体实现类，上述三个都是包装类，与四大组件的通信离不开Service。
    - ContextImpl中持有packageInfo。一般 Android 通过 PackageInfo 这个类来获取应用安装包信息，比如应用内包含的所有 Activity 名称、应用版本号之类的。PackageInfo 通过 PackageManager 来获取， PM是系统类无法hook，往往hook PackageParser

#### 启动Service流程
- APP 通知 AMS 启动Service信息
- AMS 检查Service进程是否存在，若不存在，则创建新进程。
- 新进程创建完毕 通知 AMS 已经ready。
- AMS 把缓存Service信息发送给新进程
- 新进程启动Service

具体过程十分类似于Activity。

绑定Service过程：
- ContextImpl 通知 AMS 绑定。
- AMS 通知到对应的 ActivityThread 进行绑定。

[参考这个博客](https://blog.csdn.net/zhourui_1021/article/details/105644942)

#### 启动广播流程
> 广播分为静态广播和动态广播，静态广播需要在配置文件中注册，他的时效性更早，可以在初始化app Launcher之前，注册广播监听。（具体时机：在安装app时，PMS会解析四大组件配置信息）

- 注册广播
    - Activity 注册广播，通过ContextImpl 通知 AMS。
    - AMS根据filter等数据，保存reciver信息（维持一个列表）

- 发送广播
    - Service -> ContextImpl 通知 AMS。
    - AMS遍历list进行发送。
    - Reciver所在进程接收到消息后，发送到主线程的消息队列中。

#### 启动ContentProvider流程
> 跨进程通信中，contentprovider通过唯一标识符`URI`进行标识。

> 他可以快速实现跨进程，大数据量级通信，原理是`匿名共享内存ASM`,即B需要A的数据，B通过URI告诉A在某一块共享内存中写数据，A写完后，B在此内存中读取。
 
> ContentProviderb本质是数据库，他需要实现CRUD。

- APP2 通知AMS 需要访问APP1的CP
- AMS 检查 APP1进程是否启动，若未启动，先开启进程，再返回代理对象。
- App2 通过APP1的代理对象，调用增删改查四大数据。

### 类加载机制
不赘述，介绍两个重要loader
- PathClassLoader 只能加载已安装apk的dex
- DexClassLoader 可以加载SDK中的文件dex