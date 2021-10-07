---
layout:     post
title:      webview性能优化
subtitle:   
date:       2021-09-30
author:     Ann
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Android
    - gradle
---
> 引言  
本章主要汇总一些webview加载h5的优化，同时针对手百T7内核，介绍其相比较原生webview的优化改进实现。最后，引入一些商业端做的优化内容以及我曾经接触过的webview相关工作（业务上的啦 哈哈哈哈）

## webview加载h5
对于一个普通用户来讲，打开一个WebView通常会经历以下几个阶段：
- 交互无反馈
- 到达新的页面，页面白屏
- 页面基本框架出现，但是没有数据；页面处于loading状态
- 出现所需的数据  

如果从程序上观察，WebView启动过程大概分为以下几个阶段：

![webview初始化流程](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2017/9a2f8beb.png)

### 无反馈状态
无响应表示这时候WebView并没有准备好，这时候是完全阻塞的状态。WebView初始化是后续其他任何任务的前提。现在普遍认为无响应态即WebView是否初始化，但本文认为在实际原生开发背景下，可将这一时间点提前到进入页面之前。
- 进入webview，intent的加载时长大概在30ms-60ms间, 如果有耗时的操作时间可能会更长,比如页面初始化包含UI的初始化等等。
- webview初始化，加载浏览器内核。参考[美团webview性能分析](https://tech.meituan.com/2017/06/09/webviewperf.html)，此步骤大概耗时几百毫秒。

### 白屏状态
游览器内核初始化已完成，开始建立连接请求url对应的主html页面以及相关css样式，js资源，并开始边加载边渲染，直到界面上开始出现内容的这段时间。

这个时间通过人眼可不好计算，幸好W3c提供的navigation Timing交互性能监测指标可以帮助我们量化这个指标参数。下面我们就来看一下这幅图1.2(图片来源网络)。

![白屏性能](https://upload-images.jianshu.io/upload_images/13129564-2e425c660b1c546d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

具体文字定义说明（参考来自网上）：

>  navigationStart 加载起始时间  
redirectStart 重定向开始时间  
redirectEnd 重定向结束时间  
fetchStart 浏览器发起资源请求时间  
domainLookupStart 查询DNS的开始时间
domainLookupEnd 查询DNS的结束时间
connectStart 开始建立TCP请求的时间
secureConnectionStart  如果在进行TLS或SSL，则返回握手时间  
connectEnd 完成TCP链接的时间  
requestStart 发起请求的时间  
responseStart 服务器开始响应的时间  
domLoading  开始渲染dom的时间  
domInteractive 未知  
domContentLoadedEventStart 开始触发DomContentLoadedEvent事件的时间  
domContentLoadedEventEnd DomContentLoadedEvent事件结束的时间  
domComplete dom渲染完成时间  
loadEventStart 触发load的时间  
loadEventEnd load事件执行完的时间

### loading状态
加载状态是页面已经开始渲染，人眼已经可见页面轮廓和部分元素。从图1.1可以看出，加载状态并不是等到整个html或是css、js等加载完成，而是加载到一部分就开始渲染，即边加载边渲染。页面加载时间同样可以通过W3c的navigation Timing获取。

## 主流提速方案

### DNS采用和客户端API相同的域名
DNS会在系统级别进行缓存，对于WebView的地址，如果使用的域名与native的API相同，则可以直接使用缓存的DNS而不用再发起请求图片。

以美团为例，美团的客户端请求域名主要位于`api.meituan.com`，然而内嵌的WebView主要位于 `i.meituan.com`。

当我们初次打开App时：

客户端首次打开都会请求`api.meituan.com`，其DNS将会被系统缓存。
然而当打开WebView的时候，由于请求了不同的域名，需要重新获取`i.meituan.com`的IP。
根据上面的统计，至少10%的用户打开WebView时耗费了60ms在DNS上面，如果WebView的域名与App的API域名统一，则可以让WebView的DNS时间全部达到1.3ms的量级。

`静态资源同理`，最好与客户端的资源域名保持一致。

### 采用分块传输
![](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2017/4c33eae7.png)
- 如果采用普通方式输出页面，则页面会在服务器请求完所有API并处理完成后开始传输。浏览器要在后端所有API都加载完成后才能开始解析。
- 如果采用chunk-encoding: chunked，并优先将页面的静态部分输出；然后处理API请求，并最终返回页面，可以让后端的API请求和前端的资源加载同时进行。
- 两者的总共后端时间并没有区别，但是可以提升首字节速度，从而让前端加载资源和后端加载API不互相阻塞。

### 并行加载
将webview初始化和主url html请求分开  

抽离出webview的代理类SonicSession(ui无关），从而可单开线程专门用来请求主html文件和缓存等相关工作。webview与SonicSession相互协作，从而实现并行处理缩短耗时。  

sonic框架的核心在于提供了一套webview与SonicSession通信协作机制来处理相互之间复杂的状态流程。比如webview是否初始化完成、主html流是否下载完成，是否需要调用缓存、数据下载完成而webview还是初始好怎么处理，初始化完成而数据还只下载一部分该如果处理都是sonic需要处理的主要关键节点。

## 手百T7内核


## 商业优化工作 & 思路

## 手百下载功能交互（hybird框架）