---
title: Ubuntu&Windows下终端挂代理方法（QV2ray）
date: 2022-02-21
tags: [Linux, Windows]
categories: [Coding]
---

## 问题
费力巴拉的在Ubuntu上装好了QV2ray（通过snap安装），发现shell还是不能用`git clone`，研究了一下，明白了。

## Ubuntu方法
打开QV2ray，点击首选项，入站设置，记住监听地址`127.0.0.1`和SOCKS设置中的端口（或者HTTP设置中的端口）。  
启动你订阅的链接。  
打开终端，输入（我的http端口是8889）  
```shell
# http
export https_proxy="http://127.0.0.1:8889"
export http_proxy="http://127.0.0.1:8889"

# 或者用socks5
export http_proxy="socks5://127.0.0.1:xxxx"
export https_proxy="socks5://127.0.0.1:xxxx"

# 测试一下
curl cip.cc
# 看到正确的代理就可以了
```
*PS：这个方法在每次使用shell时都要搞一次*

永久生效方法：
打开`~/.bashrc`配置文件，在最后添加如下内容，重启终端。
```shell
export https_proxy="http://127.0.0.1:8889"
export http_proxy="http://127.0.0.1:8889"
```

登录Ubuntu自动打开Qv2ray的方法：
1. 商店搜索`alacarte`，下载一个名为`main menu`的软件；
2. 打开`main menu`，找到`Qv2ray`，查看详情，复制`Command`项；
3. 打开`Startup Applications Preferences`软件；
4. add，将刚刚复制的`Command`粘贴进去即可。


## windows方法
cmd：  
```shell 
# http
set https_proxy=http://127.0.0.1:xxxx
set http_proxy=http://127.0.0.1:xxxx

# 测试
curl https://api.myip.com

# curl cip.cc 在cmd里面好像不行
ftp cip.cc
```

powershell：（需要powershell7以上版本）  
```shell
# 临时代理
$env:http_proxy="http://127.0.0.1:xxxx"
$env:https_proxy="http://127.0.0.1:xxxx"

# 查看代理
$env:http_proxy
$env:https_proxy
```

设置永久代理：  
电脑属性->选择高级系统设置->选择环境变量->选择系统变量->选择新建新建https_proxy和http_proxy，值设置为http://127.0.0.1:8889。