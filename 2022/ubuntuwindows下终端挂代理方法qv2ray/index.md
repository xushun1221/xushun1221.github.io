# Ubuntu&Windows下终端挂代理方法（QV2ray）


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

## windows方法
cmd：  
```shell 
# http
set https_proxy=http://127.0.0.1:xxxx
set http_proxy=http://127.0.0.1:xxxx

# 测试
curl https://api.myip.com

# cip.cc 在cmd里面好像不行
ftp cip.cc
```

写个批处理脚本自动开始写bolg  
```shell
@echo on
start cmd /k "set http_proxy=http://127.0.0.1:10809 & set https_proxy=http://127.0.0.1:10809 & curl https://api.myip.com & cd "D:\Blog\xushun1221.github.io" & code ./content"
```

powershell：  
```shell
$env:http_proxy="http://127.0.0.1:xxxx"
$env:https_proxy="http://127.0.0.1:xxxx"
```
