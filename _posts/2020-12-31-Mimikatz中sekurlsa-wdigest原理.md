---
title: Mimikatz sekurlsa::wdigest原理
date: 2020-12-31 23:59:45 +0800
categories: [域安全, TA0006凭据访问]
tags: [mimikatz, TA0006凭据访问]     # TAG names should always be lowercase
typora-root-url: ..
---



# Mimikatz sekurlsa::wdigest原理

看了以下几篇文章，整理一下笔记:

[Mimikatz中sekurlsa::wdigest的实现](https://3gstudent.github.io/3gstudent.github.io/Mimikatz%E4%B8%ADsekurlsa-wdigest%E7%9A%84%E5%AE%9E%E7%8E%B0/)

[Exploring Mimikatz - Part 1 - WDigest](https://blog.xpnsec.com/exploring-mimikatz-part-1/)

[深入分析Mimikatz：WDigest](https://www.anquanke.com/post/id/180126)



## 简介

wdigest.dll是LSASS中实现Digest Authentication的SSP，LSASS会将加密后的密码传给它，但是可以拿到密钥并解密，Mimikatz sekurlsa::wdigest就实现了这一过程。涉及凭据获取的关键步骤如下:

1. lsasrv!LsaInitializeProtectedMemory 每次LSASS启动时生成随机的密钥
	根据不同Windows版本，此函数固定偏移的地方可以找到存储AES密钥和3Des密钥的结构体；
	
2. WDigest!SpAcceptCredentials 是一个回调函数，被LSASS调用以接收明文凭据用于身份验证
        1. 执行lsasrv!LsaProtectMemory 加密凭据，它是作为一个CFG stub函数实现的。
                	根据待加密的数据块长度来选择加密/解密函数，AES或3Des
            2. wdigest!LogSessHandlerPasswdSet里引用了l_LogSessList指针，它是一个缓存了登录用户名及加密凭据的链表。

![t01a9c64173fa2aa9ac](/assets/img/t01a9c64173fa2aa9ac.png)

所以获取明文密码的过程分以下几步:

1. 提升至Debug权限，以访问Lsass进程
2. 枚举进程列表，获取LSASS进程句柄
3. 枚举LSASS进程加载模块列表，获取lsasrv.dll、wdigest.dll地址
4. 通过signature搜索从lsasrv.dll中定位lsasrv!LsaInitializeProtectedMemory，找到存储密钥的结构体，提取密钥
5. 通过signature搜索从wdigest.dll中定位l_LogSessList，读取加密凭据后解密



## 实现细节

见`CPlusProject\sekurlsa-wdigest`

> https://github.com/3gstudent/Homework-of-C-Language/blob/master/sekurlsa-wdigest.cpp
>
> 代码实现了对64位系统的凭据读取，输出信息同mimikatz的sekurlsa::wdigest结果相同，支持以下操作系统：
>
> - Win7 x64/Windows Server 2008 x64/Windows Server 2008R2 x64
> - Win8 x64/Windows Server 2012 x64/Windows Server 2012R2 x64
> - Win10_1507(and before 1903) x64
>
> 如果想要支持Win10_1903，可添加对Win10_1903及更高版本的系统进行识别，加上对应的偏移计算即可
>
> 如果想要支持32位系统，修改对应变量的偏移即可



对于Windows Server 2008 R2及更高版本的系统，默认配置下无法在凭据中保存明文信息，也就无法导出明文口令，可通过修改注册表启用Wdigest Auth解决这个问题，方法如下：

```
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f
```