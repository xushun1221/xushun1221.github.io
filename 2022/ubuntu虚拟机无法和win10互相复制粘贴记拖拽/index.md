# Ubuntu虚拟机无法和win10互相复制粘贴记拖拽


## 问题
Ubuntu虚拟机无法和win10系统互相复制粘贴以及拖拽文件。安装vmware-tools无用。

## 解决方法
1. 不要安装vmware-tools，如安装过，先卸载
2. `sudo apt-get autoremove open-vm-tools`
3. `sudo apt-get install open-vm-tools-desktop`
4. 重启。
