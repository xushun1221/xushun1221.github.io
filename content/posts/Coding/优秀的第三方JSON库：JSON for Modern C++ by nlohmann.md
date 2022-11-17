---
title: 优秀的第三方JSON库：JSON for Modern C++ by nlohmann
date: 2022-11-17
tags: [C++, Linux, JSON]
categories: [Projects]
---

## JSON 是啥

Json是一种轻量级的数据交换格式（也叫数据序列化方式）。Json采用完全**独立于编程语言**的文本格式来存储和表示数据。简洁和清晰的层次结构使得 Json 成为理想的数据交换语言。 易于人阅读和编写，同时也易于机器解析和生成，并有效地提升网络传输效率。

JSON的效率高于xml，低于protobuf，但学习和使用成本较低。


## nlohmann - JSON for Modern C++

感谢德国大牛：[nlohmann](https://github.com/nlohmann/json)

该库具有以下特点：

- 直观的语法
- 整个代码由一个头文件组成 json.hpp，没有子项目，没有依赖关系，没有复杂的构建系统，使用起来非常方便
- 使用 C++ 11 标准编写
- 使用 json 像使用 STL 容器一样
- STL 和 json 容器之间可以相互转换
- 严谨的测试：所有类都经过严格的单元测试，覆盖了 100% 的代码，包括所有特殊的行为。此外，还检查了 Valgrind 是否有内存泄漏。为了保持高质量，该项目遵循核心基础设施倡议(CII)的最佳实践


## 使用JSON

这个库只有一个头文件，从GitHub上down下来就行，使用时包含头文件即可。


### 序列化  示例1

```cpp
#include "json.hpp"
using json = nlohmann::json;

#include <iostream>
#include <vector>
#include <map>
#include <string>
using namespace std;

// 序列化1
void func1() {
    json js; // json对象
    // 插入键值对
    js["msg_type"] = 2;
    js["from"] = "zhangsan";
    js["to"] = "lisi";
    js["msg"] = "hello, how r u";
    cout << js << endl;
    // 序列化
    string sendBuf = js.dump();
    cout << sendBuf.c_str() << endl;
}

// 序列化2
void func2() {
    json js;
    // 添加数组
    js["id"] = {1, 2, 3, 4, 5};
    // 添加键值对
    js["name"] = "zhansan";
    // 添加对象
    js["msg"]["zhangsan"] = "hello";
    js["msg"]["lisi"] = "hi";
    // 上面两句等同于这一句
    js["msg"] = {{"zhangsan", "hello"}, {"lisi", "hi"}};
    cout << js << endl;
    string sendBuf = js.dump();
    cout << sendBuf.c_str() << endl;
}

int main(int argc, char** argv) {
    func1();
    func2();
    return 0;
}
```

输出  
```console
[Running] cd "/home/xushun/testjson/" && g++ testjson.cc -o testjson && "/home/xushun/testjson/"testjson
{"from":"zhangsan","msg":"hello, how r u","msg_type":2,"to":"lisi"}
{"from":"zhangsan","msg":"hello, how r u","msg_type":2,"to":"lisi"}
{"id":[1,2,3,4,5],"msg":{"lisi":"hi","zhangsan":"hello"},"name":"zhansan"}
{"id":[1,2,3,4,5],"msg":{"lisi":"hi","zhangsan":"hello"},"name":"zhansan"}

[Done] exited with code=0 in 1.091 seconds
```

### 序列化  示例2 - 容器

```cpp
#include "json.hpp"
using json = nlohmann::json;

#include <iostream>
#include <vector>
#include <map>
#include <string>
using namespace std;

// 序列化 容器
void func3() {
    json js;
    // vector
    vector<int> vec{1, 3, 5};
    js["list"] = vec;
    // map
    map<int, string> mp;
    mp.insert({1, "xiaoming"});
    mp.insert({2, "awei"});
    mp.insert({3, "dashan"});
    js["user"] = mp;
    cout << js << endl;
}

int main(int argc, char** argv) {
    // func1();
    // func2();
    func3();
    return 0;
}
```

输出  
```console
[Running] cd "/home/xushun/testjson/" && g++ testjson.cc -o testjson && "/home/xushun/testjson/"testjson
{"list":[1,3,5],"user":[[1,"xiaoming"],[2,"awei"],[3,"dashan"]]}

[Done] exited with code=0 in 1.168 seconds
```



### 反序列化  示例1

```cpp
#include "json.hpp"
using json = nlohmann::json;

#include <iostream>
#include <vector>
#include <map>
#include <string>
using namespace std;

// 反序列化1
void func4() {
    json js;
    js["msg_type"] = 2;
    js["from"] = "zhangsan";
    js["to"] = "lisi";
    js["msg"] = "hello, how r u";
    string sendBuf = js.dump();

    // 反序列化
    string recvBuf = sendBuf;
    json js2 = json::parse(recvBuf);
    cout << js2 << endl;
    cout << js2["msg_type"] << endl;
    cout << js2["from"] << endl;
    cout << js2["to"] << endl;
    cout << js2["msg"] << endl;
}

// 反序列化2
void func5() {
    json js;
    js["id"] = {1, 2, 3, 4, 5};
    js["name"] = "zhansan";
    js["msg"] = {{"zhangsan", "hello"}, {"lisi", "hi"}};
    string sendBuf = js.dump();

    // 反序列化
    string recvBuf = sendBuf;
    json js2 = json::parse(recvBuf);
    cout << js2 << endl;
    // 数组
    auto arr = js2["id"];
    for (auto i : arr) {
        cout << i << " ";
    } cout << endl;
    // 键值对
    cout << js2["name"] << endl;
    // json对象
    json jsmsg = js2["msg"];
    cout << jsmsg["zhangsan"] << endl;
    cout << jsmsg["lisi"] << endl;
}


int main(int argc, char** argv) {
    // func1();
    // func2();
    // func3();
    // func4();
    func5();
    return 0;
}
```

输出  
```console
[Running] cd "/home/xushun/testjson/" && g++ testjson.cc -o testjson && "/home/xushun/testjson/"testjson
{"id":[1,2,3,4,5],"msg":{"lisi":"hi","zhangsan":"hello"},"name":"zhansan"}
1 2 3 4 5 
"zhansan"
"hello"
"hi"

[Done] exited with code=0 in 1.447 seconds
```

### 反序列化  示例2 - 容器

```cpp
#include "json.hpp"
using json = nlohmann::json;

#include <iostream>
#include <vector>
#include <map>
#include <string>
using namespace std;

// 反序列化 容器
void func6() {
    json js;
    vector<int> vec{1, 3, 5};
    js["list"] = vec;
    map<int, string> mp;
    mp.insert({1, "xiaoming"});
    mp.insert({2, "awei"});
    mp.insert({3, "dashan"});
    js["user"] = mp;
    string sendBuf = js.dump();
    
    // 反序列化
    string recvBuf = sendBuf;
    json js2 = json::parse(recvBuf);
    cout << js2 << endl;
    // 数组
    vector<int> vec2 = js2["list"];
    for (auto i : vec2) {
        cout << i << " ";
    } cout << endl;
    // map
    map<int, string> mp2 = js2["user"];
    for (auto pr : mp2) {
        cout << pr.first << ":" << pr.second << endl;
    }
}


int main(int argc, char** argv) {
    // func1();
    // func2();
    // func3();
    // func4();
    // func5();
    func6();
    return 0;
}
```

输出  
```console
[Running] cd "/home/xushun/testjson/" && g++ testjson.cc -o testjson && "/home/xushun/testjson/"testjson
{"list":[1,3,5],"user":[[1,"xiaoming"],[2,"awei"],[3,"dashan"]]}
1 3 5 
1:xiaoming
2:awei
3:dashan

[Done] exited with code=0 in 1.54 seconds
```