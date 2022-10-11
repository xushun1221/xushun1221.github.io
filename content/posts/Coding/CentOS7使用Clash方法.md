---
title: CentOS7使用Clash方法
date: 2022-10-11
tags: [Linux, CentOS]
categories: [Coding]
---

## 安装使用步骤

1. 用户目录下创建`clash`文件夹
2. [clash-releases](https://github.com/Dreamacro/clash/releases)下载Clash的二进制文件到`~/clash`文件夹，下载这个就行`clash-linux-amd64-v1.11.8.gz`，解压后重命名为`clash`
3. 在该目录下下载配置文件，`[xushun@localhost clash]$ wget -O config.yaml 你的配置文件url`
4. 启动程序，会下载`Country.mmdb`文件，关闭程序
    ```shell
    [xushun@localhost clash]$ chmod +x clash 
    [xushun@localhost clash]$ ./clash -d .
    ```
5. 在`config.yaml`配置文件开头添加端口号，保存
    ```
    port: 9876 # for http(s)
    socks-port: 9875 # for socks5
    ```
6. 打开系统网络设置->VPN->Network Proxy->设置为Manual->将刚刚设置的端口号填进去
7. 再启动clash`./clash -d .`
8. 用另一个终端测试：`curl https://api.myip.com`，测试成功


## 设置开机启动

1. 将配置文件`cache.db  config.yaml  Country.mmdb`移动到`/etc/clash`
2. 创建service文件：`sudo touch /etc/systemd/system/clash.service`
3. 编辑service文件：
    ```
    [Unit]
    Description=clash daemon

    [Service]
    Type=simple
    User=root
    ExecStart=/home/xushun/clash/clash -d /etc/clash/
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```
4. 重新加载systemctl daemon：`sudo systemctl daemon-reload`
5. 启动clash：`sudo systemctl start clash.service`
6. 设置clash开机启动：`sudo systemctl enable clash.service`

几个常用命令：  
```
启动Clash
sudo systemctl start clash.service
重启Clash
sudo systemctl restart clash.service
查看Clash运行状态
sudo systemctl status clash.service
```

## 更新订阅

1. 先关闭服务`sudo systemctl stop clash.service`
2. `unset https_proxy`
3. `wget -O config.yaml 你的订阅url`
4. `config.yaml`中添加端口号
5. 把它复制到`/etc/clash`
6. 重启服务`sudo systemctl restart clash.service`