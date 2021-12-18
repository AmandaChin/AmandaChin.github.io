---
layout:     post
title:      git多账号配置
subtitle:   
date:       2021-11-21
author:     Ann
header-img: img/bg-street.jpg
catalog: true
tags:
    - 环境配置
---

> 以配置github环境为示例。

- Step 1

查看是否设置过全局用户名
```
git config --list
```
如果设置过 用下面命令unset掉
```
git config --global --unset user.name
git config --global --unset user.email
```

- Step 2

在`~/.ssh`目录下生成对应的秘钥，使用命令`ssh-keygen -t rsa -C "邮箱"`
```
ssh wangcheng$ ssh-keygen -t rsa -C "***@163.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/wangcheng/.ssh/id_rsa): id_rsa_github
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa_github.
Your public key has been saved in id_rsa_github.pub.
The key fingerprint is:
SHA256:ZXQZ0CmX9SQ/SQbXmjH3VQ1Ng3yxvKPXPuBGqI8mMIg ***@163.com
The key's randomart image is:
+---[RSA 2048]----+
|          oo+B*XO|
|         ...*oB*X|
|          oo  .@=|
|         o    o +|
|  . .   S   .  o |
| E . o     . o. o|
|      o   . o....|
|       . o.  o.o |
|        o....   o|
+----[SHA256]-----+
```
复制`id_rsa_github.pub`公钥到github的ssh配置中。

- Step 3

添加私钥
```
ssh-add ~/.ssh/私钥文件名
```

- Step 4
在`~/.ssh`目录下，配置config文件

``` shell
# github 
Host github
HostName github.com
User AmandaChin
IdentityFile ~/.ssh/id_rsa_github
# 配置文件参数
# Host : Host可以看作是一个你要识别的模式，对识别的模式，进行配置对应的的主机名和ssh文件
# HostName : 要登录主机的主机名
# User : 登录名
# IdentityFile : 指明上面User对应的identityFile路径
```

- Step 5

测试是否连接成功
```
SSH -T git@github.com
```

- Step 6

具体文件提交代码时，可以配置local属性
```
git config --local user.name "***"
git config --local user.email   
```

## 问题更新
### 本地无法打开github or git操作显示timedOut
- ping github.com看下访问是否正常
- 修改host中的IP地址(https://ipaddress.com/website/github.com)

[查看IP地址参考](https://blog.csdn.net/qq_32563773/article/details/106273247)  
[修改mac host](https://www.jianshu.com/p/0260cbc51687)

- ip地址会变，遇到此问题可以先试下注释掉host中的配置