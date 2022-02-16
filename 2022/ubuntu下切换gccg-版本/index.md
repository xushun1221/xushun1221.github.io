# Ubuntu下切换gccg++版本


## 前言
我的`Ubuntu18.04`安装的`build-essential`默认的`gcc/g++`版本为7.5，现在需要8版本的，切换版本的操作如下。

## 操作

查看`gcc g++`版本  
```shell
ls /usr/bin/gcc*
ls /usr/bin/g++*
```

安装`gcc-8, g++-8`  
```shell
sudo apt install gcc-8 g++-8
```

切换默认版本（将要用的编译器优先级设置高一点）  
```shell
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 80
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 70

sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-8 80
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 70
```

也可以直接删了旧版本，然后为新版本创建一个软连接
```shell
sudo apt remove gcc-7 g++-7

sudo rm /usr/bin/gcc
sudo rm /usr/bin/g++
sudo ln -s /usr/bin/gcc-8 /usr/bin/gcc
sudo ln -s /usr/bin/g++-8 /usr/bin/g++
```


