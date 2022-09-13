# 【C++高级】04 - c++11总结和c++多线程


 ## c++11标准内容总结

 1. 关键字和语法
    1. `auto`：可以根据右值推导出右值的类型，确定左边变量的类型；
    2. `nullptr`：原来使用的`#define NULL 0`是宏定义，`nullptr`是指针专用，能区别指针和整数；
    3. `for(Type val : container) { }`：可以遍历数组和容器等，底层就是指针或迭代器遍历；
    4. 右值引用：在对象优化中作用巨大，`move`移动语义，`forward`类型完美转发；
    5. `typename...A`：模板的一个新特性，表示可变参数。
 2. 绑定器和函数对象
    1. `function`函数对象；
    2. `bind`绑定器；
    3. `lambda`表达式。
 3. 智能指针
    1. `shared_ptr`强智能指针；
    2. `weak_ptr`弱智能指针。
4. 容器
   1. `unordered_set`和`unordered_map`：新的哈希表容器；
   2. `array`：数组容器；
   3. `forward_list`：前向链表。
5. c++多线程编程
   1. c++11支持了语言级别的多线程编程。


## c++11 语言级别的多线程

c++11 支持了语言级别的多线程编程，最大的优点就是跨平台。

它底层调用的仍然是，操作系统提供的系统调用API，如Linux下使用`pthread`线程库。

主要学习内容：  
- `thread`
- `mutex`
- `condition_variable`
- `lock_quard`
- `unique_lock`
- `atomic`
- `sleep_for`

使用c++11多线程，需要包含`thread`头文件。



## 通过 thread 类编写c++多线程程序

1. 创建启动线程：`std::thread`定义一个线程对象，传入线程函数和所需的参数，线程自动开启运行；
2. 子线程结束：子线程函数运行完成，线程就结束了；
3. 主线程处理子线程结束：
   1. c++在语言级别对线程结束进行了限制，当主线程运行完成，还有未完成的子线程，该进程就会异常终止；
   2. `join()`：如`t1.join();`，阻塞等待某个线程结束；
   3. `detach()`：如`t1.detach();`，将某个线程设置为分离线程，与主线程断开联系，无需主线程`join`。当主线程结束，进程就结束了，所有的子线程都自动结束。
  
示例：  
```C++
#include <iostream>
#include <thread>
using namespace std;
void threadHandle1(int sleeptime) {
    /* 
        std::this_thread 命名空间中定义了一些对当前线程执行操作的函数 
        std::chrono 命名空间中定义了一些与时间相关的函数
    */
    std::this_thread::sleep_for(std::chrono::seconds(sleeptime)); /* 睡眠两秒 */
    cout << "hello thread1" << endl;
}
void threadHandle2(int sleeptime) {
    std::this_thread::sleep_for(std::chrono::seconds(sleeptime)); /* 睡眠三秒 */
    cout << "hello thread2" << endl;
}
int main() {
    /* 定义一个线程对象 传入一个线程函数 和 参数 */
    thread t1(threadHandle1, 2);
    thread t2(threadHandle2, 3);
    /* 主线程阻塞等待 t1 线程退出 */
    t1.join();
    t2.join();
    /* 可以将子线程设置为 分离线程 */
    // t1.detach();
    cout << "main thread done" << endl;
    /* 这是语言级别的限制 */
    /* 主线程执行完成 如果当前进程还有未完成的子线程 该进程就会异常终止 */
    return 0;
}
```


## 线程间互斥 - mutex、lock_guard、unique_lock

使用互斥锁，需要包含头文件`mutex`。

`mutex`使用示例：  
```C++
#include <iostream>
#include <thread>
#include <list>
#include <mutex>
using namespace std;

/* 车站一共100张车票 三个窗口一起卖票 */
int ticket_count = 100;
/* 全局互斥锁 */
std::mutex mtx;
/* 模拟卖票的线程函数 */
void sellTicket(int index) { 
    while (ticket_count > 0) {
        /* 访问共享数据前加锁 阻塞加锁 */
        mtx.lock();
        /* 防止 ticket_count 已经为 0 但仍然进入循环 需要再次判断 */
        if (ticket_count > 0) {
            cout << index << "窗口卖出第" << ticket_count -- << "张票" << endl;
        }
        /* 解锁 */
        mtx.unlock();
        std::this_thread::sleep_for(std::chrono::microseconds(100));
    }
}

int main() {
    list<std::thread> tlist;
    for (int i = 1; i <= 3; ++ i) {
        tlist.push_back(std::thread(sellTicket, i));
    }
    for (std::thread& t : tlist) {
        t.join();
    }
    cout << "sell done" << endl;
    return 0;
}
```

使用`mutex`需要在访问共享数据后进行解锁操作，还可以使用`lock_guard`来管理互斥锁，它会在出作用域时自动释放互斥锁。

`lock_guard`使用示例：  
```C++
#include <iostream>
#include <thread>
#include <list>
#include <mutex>
using namespace std;
int ticket_count = 100;
std::mutex mtx;
void sellTicket(int index) { 
    while (ticket_count > 0) {
        {
            // mtx.lock();
            /* 使用局部lock_guard对象管理mutex锁 构造时加锁 */
            lock_guard<std::mutex> lock(mtx);
            if (ticket_count > 0) {
                cout << index << "窗口卖出第" << ticket_count -- << "张票" << endl;
            }
            // mtx.unlock();
            /* 出作用域时析构 自动解锁 */
        }
        std::this_thread::sleep_for(std::chrono::microseconds(100));
    }
}
int main() {
    list<std::thread> tlist;
    for (int i = 1; i <= 3; ++ i) {
        tlist.push_back(std::thread(sellTicket, i));
    }
    for (std::thread& t : tlist) {
        t.join();
    }
    cout << "sell done" << endl;
    return 0;
}
```


注意，`lock_guard`不支持拷贝构造和赋值，所以只能用在比较简单的逻辑中。

`unique_lock`同样不支持左值拷贝构造和赋值，但是它支持带右值引用参数的拷贝构造和赋值，可以通过右值进行拷贝构造和赋值，可以用在产生临时对象的地方。使用`unique_lock`比直接使用`mutex`要安全很多。一般在线程通信中会用到。

`lock_guard`和`unique_lock`的区别就相当于`scoped_ptr`和`unique_ptr`的区别。



## 线程同步通信 - 生产者消费者模型

示例：  
```C++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
using namespace std;

/* 全局互斥锁 互斥操作 */
std::mutex mtx;
/* 全局条件变量 同步通信操作 */
std::condition_variable cv;
/* 注意 c++STL 中提供的容器是不保证线程安全的 我们需要对其进行封装 */
class Queue {
public:
    void put(int val) { /* 物品为空时生产物品 */
        unique_lock<std::mutex> lck(mtx); /* 加锁 */
        while (!que.empty()) {
            /* 不为空时 生产者进入等待状态 解锁 */
            cv.wait(lck); /* 解锁lck 等待条件变量满足时 尝试加锁（阻塞） */
        }
        que.push(val);
        cout << "生产者：生产" << val << "号物品" << endl;
        /* 生产完成需要通知消费者进行消费 */
        cv.notify_all(); /* 通知其他所有线程 */
        /* 出作用域 自动解锁 */
    }
    int get() { /* 物品不为空时消费物品 */
        unique_lock<std::mutex> lck(mtx); /* 加锁 */
        while (que.empty()) {
            /* 物品为空时 进入等待状态 */
            cv.wait(lck); /* 解锁 等待条件变量 尝试加锁 */
        }
        int val = que.front();
        que.pop();
        cout << "消费者：消费" << val << "号物品" << endl;
        /* 消费完成通知生产者进行生产 */
        cv.notify_all();
        return val;
        /* 出作用域 自动解锁 */
    }
private:
    queue<int> que;
};

void producer(Queue* que) {
    /* 生产10个物品 */
    for (int i = 1; i <= 10; ++ i) {
        que->put(i);
        std::this_thread::sleep_for(std::chrono::seconds(rand() % 3));
    }
}
void consumer(Queue* que) {
    /* 消费10个物品 */
    for (int i = 1; i <= 10; ++ i) {
        que->get();
        std::this_thread::sleep_for(std::chrono::seconds(rand() % 3));
    }
}
int main() {
    /* 共享的队列 */
    Queue que;
    std::thread t1(producer, &que);
    std::thread t2(consumer, &que);
    t1.join();
    t2.join();
    return 0;
}
```



## 基于 CAS 操作的 atomic 原子类型

头文件`atomic`。

对于一些类似于`++  --`的简单操作，没有必要使用互斥锁来保证线程安全，可以使用 CAS 操作的原子特性保证线程安全。


示例：  
```C++
#include <iostream>
#include <thread>
#include <atomic>
#include <list>
using namespace std;
volatile std::atomic_bool isReady{false}; /* volatile 防止共享数据被线程缓存 */
volatile std::atomic_int mycount{0};
void task() {
    while (!isReady) {
        /* 放弃当前时间片 */
        std::this_thread::yield();
    }
    for (int i = 0; i < 100; ++ i) {
        ++ mycount;
    }
}
int main() {
    list<std::thread> tlist;
    for (int i = 0; i < 10; ++ i) {
        tlist.push_back(std::thread(task));
    }
    std::this_thread::sleep_for(std::chrono::seconds(2));
    isReady = true;
    std::this_thread::sleep_for(std::chrono::seconds(2));
    cout << mycount << endl;
    for (std::thread& t : tlist) {
        t.join();
    }
    return 0;
}
```









