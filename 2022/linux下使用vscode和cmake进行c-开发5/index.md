# Linux下使用VSCode和CMake进行C++开发【5】


## 前言
学了这么多，写一个简单的项目来总结一下。

## 写源码
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【5】/编写源码.jpg "源码")

-----

## 编译
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【5】/cmake编译.jpg "编译")

-----

## 调试
点击Run and Debug按钮，新建`launch.json`文件，然后按`F5`即可开始调试，在此之前，把`CMakeLists.txt`改一下：  
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【5】/cmakelists.jpg "cmakelists")

`launch.json`内容：  
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【5】/调试前launchjson.jpg "launch")

调试完成后，把`launch.json`文件中的`preLaunchTask`选项取消注释，然后新建一个`tasks.json`文件，`launch.json`和`tasks.json`内容如下：  
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【5】/tasks.jpg "tasks")

然后就无需手动编译，直接`F5`即可自动完成编译并进入调试。



## 参考资料
1. https://www.bilibili.com/video/BV1fy4y1b7TC
