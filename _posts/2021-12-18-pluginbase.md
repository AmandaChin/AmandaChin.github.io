---
layout:     post
title:      四大组件插件化基本原理
subtitle:   
date:       2021-12-18
author:     Ann
header-img: img/bg_eden.jpeg
catalog: true
tags:
    - Android
    - 插件化
---
> 介绍《Android插件化开发指南》中通用插件化方案

## Activity
### 对启动Activity行为hook
假设我们需要在启动Activity过程中落日志记录，有以下几种方法：
- 构建通用基类BaseActivity，重写startActivityForResult方法
- startActivity前半段：反射仪表盘mInstrumentation（仅对当前Activity生效，反射内容复杂）
- hook AMN.getDefault()方法 （全局生效，一劳永逸，常规方案。hook时机常在Application attachBaseContext中）
- startActivity后半段：反射仪表盘mInstrumentation
- hook ActivityThread中的H.callback方法

### 如何启动没有在配置文件中声明的Activity？
> 整体思想：不占坑方式（宿主声明几个占位Activity）,通过hook方式欺骗AMS校验规则。占坑方式（宿主声明***FackActivity对应插件 ***Activity）,仍需通过hook方式欺骗AMS校验规则
- 方案1：为每个插件创建一个ClassLoader
ActivityThread.performLaunchActivity方法如下：

```java
    private Activity performLaunchActivity() {
        Activity activity  = null;
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(cl,component.getClassName(),r.intent);
        return activity;
    }
```
    
    在执行newActivity方法时需要制定classLoader，这个classLoader是通过LoadedApk（r.packageInfo）读取出来的，他是一个缓存的对象。

    对于插件，可以构建`插件LoadedApk`，并把它放到`mPackages`缓存中。然后反射得到插件LoadedApk对象的classloader，设置为插件的类加载器。

    - 总结来说：此方案需要反射各种类型对象，还需适配Android版本，十分繁琐，且危险。

- 方案2：合并多个dex
    - 通过宿主ClassLoader获取宿主`dexElement`字段
    - 根据插件的apkFile，反射一个`Element`对象，这是插件的dex
    - 把插件的dex合并到宿主的dexPath中，合并为新的`dexElement`字段

    注：这个方案也是热修复的思想，但是对应的资源文件的Resource也变得一起了，需要考虑资源冲突的问题

- 方案3：修改APP的原生classloader
    - hook系统classLoader，并修改类加载规则，先从宿主中查找，再从插件中查找（不重复从parent中找）。

    注：这个方案会导致Class.forName等在parent的类加载器方法失效。

    

## 资源
> 资源文件分为两类: 1. `assest`下的资源文件，不参与打包，直接通过aapt进行打入dex。 2. `res`下的可编译资源文件，layout、drawable等，会被打包成R.java

### assets下的文件
- 通过getResource().getAssets().open("filename");获取目标目录下的文件资源

Resources.getAssets()返回AssetManager对象，通过反射`addAssetPath(String path)`方法，可以把插件资源文件放入。

### res下的文件
- 通过反射对应的r文件，通过getResource().getDrawable(R.id.***);获取插件资源

### 插件资源加载方案
- 反射`AssetManager`方法，重写`addAssetPath`将插件路径添加至宿主AssetManager对象中。
- 重写Activity的getAsset、getResoucres、getTheme方法，让他优先加载我们自定义的含有插件路径的AssetManager、Resource、Theme。
- 加载外部插件，生成插件对应的DexClassLoader
- 反射获取插件方法（插件内置get**ResId()一类的方法，返回对应的资源）
- R.java下的文件也可以直接反射插件的R文件，获取对应的资源。

> ”换肤“方案，可以生成不同肤色的插件，保证接口统一，通过上述方案快速切换。

> 资源热知识： 资源ID是由`PPTTNNNN`（package+type+id自增号）组成的32字节ID，在同一个Resource下需要考虑资源冲突的问题。

如何解决资源冲突？
- 重写AAPT命令，打包过程中将插件资源加上限定前缀。
- 打包后，重写R.java & resources.arsc中对应id值，使插件资源加上限定前缀
- 在public.xml中指定apk的所有xml值（不易维护）

> 另一种方案，每个插件拥有独立的AssetManager和Resource这样就永远不会有资源冲突的问题了。在使用的过程中，根据当前使用场景动态的切换对应的Class类。


## Service
### 方案1：占坑方案，在宿主配置文件中声明N个预置Service
整体方案：
- 合并dex
- 在Service启动过程中欺骗AMS

- 如何欺骗startService
    - hook `AMN.getDefault()` 替换代理类，通知AMS加载预置Service
    - hook `H.callback` 在下半段startService时替换预置Service加载正常插件的startService方法

- 如何欺骗bindService
    - 在startService的基础上，仅需在`H.callback`处理bindService。

## BroadcaseReceiver
广播分为静态广播和动态广播，他们的发送接收原理如下：
- 发送广播
    - 通过Context.sendBroadcast方法发送广播，最终会调用到AMN.getDefault().broadcasetIntent方法，把要发送的广播传给AMS
- 接收广播
    -  AMS接收到消息后，检查PMS，AMS中维护的广播数组，看下哪些广播符合条件，通知对应的app进程启动广播（调用onReceive方法）

### 动态广播插件化方案
> 动态广播的注册信息存在于AMS中，对于动态广播，不需要写入配置文件，也无需和AMS打交道，我们仅需要保证宿主app可以加载到插件类即可（如合并dex、改classLoader等方法）

### 静态广播插件化方案
> 静态广播的注册信息存在于PMS中，动态广播因为后注册，优先级会比静态广播高，但是静态广播有不一样的意义，他的注册时机是在安装包时，因此他可以早于首页启动，先去注册一些广播监听。
- 为插件广播在宿主中创建一个展位广播StubReceiver(只需一个即可，因为可以无限扩展action)
- 通过metadata简历宿主展位action与插件action的映射关系
- 反射`packageParser`对象，获取到receivers List，及metaData中的action
- 将插件中的静态广播，手动注册为动态广播，并取出他们的action
- 将metaData中的action，和插件的action映射关系存成hashMap
- 宿主展位receiver进行分发，接收到action后从hashmap取出映射关系，并给插件发送动态广播。

## ContentProvider
ContentProvider组件，和其他三个组件不太一样的是，定义它往往是给其他app/进程使用的。相当于生了女儿，嫁给别人作媳妇。对于插件中的contentProvider，像是养女，先要过继到名下，才能嫁给别人做媳妇。

- “养女过继”
    - 合并dex
    - 读取插件中的contentProvider信息
    - 把插件中的contentProvider的packageName设为宿主中的name
    - 反射执行ActivityThread中的installContentProvider方法，把插件cp安装到宿主上。执行时机需要在Application attachBaseContext时

- ”养女出嫁“
外部app通过唯一标识URI去调用contentProvider服务，对于宿主来说主要是`分发`.

把`content://host_auth/plugin_auth/path/query`转为`content://plugin_auth/path/query`