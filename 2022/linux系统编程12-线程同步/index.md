# 【Linux系统编程】12 - 线程同步


## 同步

### 什么是同步
所谓同步，即同时起步，协调一致。不同的对象，对**同步**的理解方式略有不同。如，设备同步，是指在两个设备之间规定一个共同的时间参考；数据库同步，是指让两个或多个数据库内容保持一致，或者按需要部分保持一致；文件同步，是指让两个或多个文件夹里的文件保持一致。等等。

而编程中、通信中所说的同步与生活中大家印象中的同步概念略有差异。同字应是指协同、协助、互相配合。主旨在协同步调，按预定的**先后次序**运行。

### 什么是线程同步
线程同步，指一个线程发出某一功能调用时，在没有得到结果之前，该调用不返回。同时其它线程为保证数据一致性，不能调用该功能。

举例 1： 银行存款 5000。柜台，折：取 3000；提款机，卡：取 3000。剩余：2000

举例 2： 内存中 100 字节，线程 T1 欲填入全 1， 线程 T2 欲填入全 0。但如果 T1 执行了 50 个字节失去 cpu，T2执行，会将 T1 写过的内容覆盖。当 T1 再次获得 cpu 继续 从失去 cpu 的位置向后写入 1，当执行结束，内存中的100 字节，既不是全 1，也不是全 0。

产生的现象叫做**与时间有关的错误(time related)**。为了避免这种数据混乱，线程需要同步。

同步的目的，是为了避免数据混乱，解决与时间有关的错误。实际上，不仅线程间需要同步，进程间、信号间等等都需要同步机制。

因此，所有**多个控制流，共同操作一个共享资源**的情况，都需要同步。

### 数据混乱的原因
1. 资源共享（独享的数据则不会）
2. 调度随机（意味着数据访问会出现竞争）
3. 线程间缺乏必要的同步机制

以上 3 点中，前两点不能改变，欲提高效率，传递数据，资源必须共享。只要共享资源，就一定会出现竞争。只要存在竞争关系，数据就很容易出现混乱。

所以只能从第三点着手解决。使多个线程在访问共享资源的时候，出现互斥。

## 互斥量(锁) mutex

### 互斥量的概念
Linux中提供互斥量（mutex），也叫互斥锁。

使用互斥锁的一半流程：每个线程在对共享资源进行操作前都需要尝试加锁，成功加锁之后才能操作，操作完成后，解锁。（对同一个数据，同一个时刻，只有一个线程持有锁）

共享资源还是共享的，线程之间也还是竞争的，但是通过**锁**，就将资源的访问变成了互斥的操作，而后关于时间的错误也不会发生了。

请看示意图：  
![](/post_images/posts/Coding/【Linux系统编程】12/互斥量示意图.jpg "互斥量示意图")

假设这样一种情况：当前有三个线程（T1，T2，T3），它们都向对共享数据进行操作，分别将数据填充为`000..`，`111..`，`777..`。  
1. 此时，T1线程获得了CPU时间片，并拿到了互斥锁，它开始向共享数据中写`000..`。
2. T1的时间片结束后，此时T2线程开始执行，它要操作共享数据需要申请锁，它就会阻塞在锁上，并释放CPU。
3. 此时T3线程开始执行，
   1. 如果T3线程也去申请锁，那它也会阻塞在锁上，并释放CPU，T1重新获得CPU并开始继续填充`000..`。
   2. 如果T3线程，不去申请锁，那它可以操作共享数据吗？答案是可以。它会向共享数据内填写`777..`直到时间片结束，这样就破坏了T1线程的工作，数据出现混乱。

这个例子表明，互斥量（互斥锁）不是强制性的，它实际上是Linux系统提供的一把**建议锁**（也称协同锁）。建议在多线程访问共享数据时使用该机制，但是并没有强制限定。

所以，必须使用互斥锁来操作共享数据。

### 互斥量操作函数
和互斥锁相关的操作函数主要有以下五个：
- `pthread_mutex_init`；初始化锁；
- `pthread_mutex_destroy`：销毁锁；
- `pthread_mutex_lock`：加锁；
- `pthread_mutex_trylock`：（非阻塞）尝试加锁；
- `pthread_mutex_unlock`：解锁；
- 它们的成功返回值都为`0`，失败返回值都是错误号。

Ubuntu默认没有安装POSIX标准的文档，可以自己安装一下：`sudo apt-get install manpages-posix-dev`。

#### pthread_mutex_init / pthread_mutex_destroy
函数原型：  
```c
#include <pthread.h>

int pthread_mutex_destroy(pthread_mutex_t *mutex);
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
    const pthread_mutexattr_t *restrict attr);
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回错误号。
- `mutex`：保存创建的互斥量，或要销毁的信号量；
- `attr`：互斥量的属性，默认传`NULL`。（参阅APUE.12.4同步属性）

注，`restrict`关键字，用来限定指针变量，被该关键字限定的指针变量所指向的内存操作，必须由本指针完成，不能由其他指针修改。

使用`pthread_mutex_init(&mutex, NULL)`可以动态创建互斥锁，也可以使用宏静态创建`pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER`。

### pthread_mutex_lock / pthread_mutex_unlock
函数原型：  
```c
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回错误号。

### 理解加锁和解锁
我们可以将互斥锁**看作一个整数**（只能为`0`或`1`），当我们初始化一个互斥锁`pthread_mutex_t mute`，它的值为`1`。

- 使用`pthread_mutex_lock`尝试加锁，
  - 如果加锁不成功（`mutex == 0`），线程阻塞，直到持有该锁的其他线程解锁为止；
  - 如果成功加锁（`mutex == 1`），`mutex --`，访问共享资源。
- 使用`pthread_mutex_unlock`解锁，`mutex ++`，解锁的同时，会将阻塞在该锁上的所有线程全部唤醒，至于哪个线程先被唤醒，取决于优先级、调度等。默认情况是，先被该锁阻塞的线程，先唤醒。

### 练习 - 多个线程打印 hello world
我们希望编写这样一个程序：使用两个线程，主线程和一个子线程交替打印`hello world`这两个单词，每个线程在打印一个单词后，要sleep几秒钟，主线程大写，子线程小写。

#### 不使用互斥锁
原始程序：  
```c
void* tfunc(void* arg) {
    while (1) {
        printf("hello ");
        sleep(rand() % 3);
        printf("world\n");
        sleep(rand() % 3);
    }
    pthread_exit(NULL);
}

int main(int argc, char** argv) {
    int ret;
    pthread_t tid;
    srand(time(NULL));
    ret = pthread_create(&tid, NULL, tfunc, NULL);
    if (ret != 0) {
        fprintf(stderr, "pthread_create error : %s\n", strerror(ret));
        exit(1);
    }
    while (1) {
        printf("HELLO ");
        sleep(rand() % 3);
        printf("WORLD\n");
        sleep(rand() % 3);
    }
    ret = pthread_join(tid, NULL);
    if (ret != 0) {
        fprintf(stderr, "pthread_join error : %s\n", strerror(ret));
        exit(1);
    }
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ ./hello 
HELLO hello world
WORLD
hello HELLO world
WORLD
hello world
HELLO hello WORLD
HELLO WORLD
^C
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ 
```
测试结果，输出的内容乱七八糟，大小写的`hello world`不能完整输出，而是交叉输出，原因是没有互斥地使用STDOUT。

#### 使用互斥锁
使用互斥量进行输出：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

pthread_mutex_t mutex; // 互斥锁

void* tfunc(void* arg) {
    while (1) {
        pthread_mutex_lock(&mutex); // 加锁
        printf("hello ");
        sleep(rand() % 3);
        printf("world\n");
        pthread_mutex_unlock(&mutex); // 解锁
        sleep(rand() % 3);
        // pthread_mutex_unlock(&mutex); // ***解锁***
    }
    pthread_exit(NULL);
}

int main(int argc, char** argv) {
    int ret;
    pthread_t tid;
    srand(time(NULL));
    ret = pthread_mutex_init(&mutex, NULL); // 创建互斥锁
    if (ret != 0) {
        fprintf(stderr, "pthread_mutex_init error : %s\n", strerror(ret));
        exit(1);
    }
    ret = pthread_create(&tid, NULL, tfunc, NULL);
    if (ret != 0) {
        fprintf(stderr, "pthread_create error : %s\n", strerror(ret));
        exit(1);
    }
    while (1) {
        pthread_mutex_lock(&mutex); // 加锁
        printf("HELLO ");
        sleep(rand() % 3);
        printf("WORLD\n");
        pthread_mutex_unlock(&mutex); // 解锁
        sleep(rand() % 3);
        // pthread_mutex_unlock(&mutex); // ***解锁***
    }
    ret = pthread_join(tid, NULL);
    if (ret != 0) {
        fprintf(stderr, "pthread_join error : %s\n", strerror(ret));
        exit(1);
    }
    ret = pthread_mutex_destroy(&mutex); // 销毁互斥锁
    if (ret != 0) {
        fprintf(stderr, "pthread_destroy error : %s\n", strerror(ret));
        exit(1);
    }
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ ./mutex 
HELLO WORLD
hello world
HELLO WORLD
hello world
HELLO WORLD
HELLO WORLD
hello world
hello world
HELLO WORLD
^C
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ 
```

#### 分析
1. 定义了全局的互斥量；
2. 两个线程的`while`中，两次`printf`前后进行了加锁和解锁；
3. 如果将解锁位置挪到第二个`sleep`之后，发现线程交替打印的现象难以出现；
   - 原因：线程操作完共享资源后，本应该立即释放，但是，操作完成后，抱着锁睡眠，睡醒后又立即加锁，这两个库函数本身不会阻塞，所以在这两行代码之间失去CPU的 概率非常小，因此，另一个线程很难获得加锁的机会；
   - 结论：尽可能让锁的**粒度变小**（访问共享资源前，加锁，访问结束后，**立即**解锁）。
4. 如果主线程输出几次后就退出循环，试图销毁锁，但子线程没有释放锁，销毁操作就无法完成。

### 非阻塞加锁 pthread_mutex_trylock
函数原型：  
```c
#include <pthread.h>

int pthread_mutex_trylock(pthread_mutex_t *mutex);
```
该函数尝试加锁，成功返回`0`，如果失败，就直接返回错误号，不会阻塞。

### 死锁
是使用锁不恰当导致的一种现象。

1. 对同一个互斥量反复加锁，如果线程已经持有锁，又对该锁进行加锁，会阻塞形成死锁；
2. 如果两个线程各自持有一把锁，又请求对对方的锁进行加锁，双方都阻塞，导致死锁。

## 读写锁
读写锁，与互斥量类似，但是读写锁允许更高的并行性。其特性为：**读共享，写独占**。

### 读写锁状态
强调：读写锁**只有一把**，但是具有两种状态，
1. 读模式下加锁状态（读锁）
2. 写模式下加锁状态（写锁）

### 读写锁特性
1. 读写锁为**写锁**时，解锁前，**所有**对该锁加锁的线程都会被阻塞；
2. 读写锁为**读锁**时，
   1. 如果线程以读模式加锁，会成功；
   2. 如果线程以写模式加锁，会阻塞；
3. 读写锁为**读锁**时，如果既有以读模式加锁的线程，也有以写模式加锁的线程，那么读写锁会阻塞随后的读锁请求，优先满足写锁请求。（读锁、写锁并行阻塞，**写锁的优先级高**）

读写锁，也叫**共享-独占锁**，当读写锁以读模式锁住时，它是以共享模式锁住的；当读写锁以写模式锁住时，它是以独占模式锁住的。（读共享，写独占）

读写锁非常适用于对数据的读的次数远大于写的次数的情况。

### 读写锁的操作函数
和互斥锁类似，读写锁的操作函数主要有以下几个：  
- `pthread_rwlock_init`：创建读写锁；
- `pthread_rwlock_destroy`：销毁读写锁；
- `pthread_rwlock_rdlock`：加读锁；
- `pthread_rwlock_wrlock`：加写锁；
- `pthread_rwlock_tryrdlock`：（非阻塞）尝试加读锁；
- `pthread_rwlock_trywrlock`：（非阻塞）尝试加写锁；
- `pthread_rwlock_unlock` ：解锁；
- 返回值都是，成功返回`0`，失败返回错误码。

#### pthread_rwlock_init / pthread_rwlock_destroy
函数原型：  
```c
#include <pthread.h>

int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
    const pthread_rwlockattr_t *restrict attr);
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
```
和互斥锁使用方法相同。

动态创建读写锁`pthread_rwlock_init(&rwlock, NULL)`，静态创建读写锁`pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER`，效果相同。

#### pthread_rwlock_rdlock / pthread_rwlock_tryrdlock
函数原型：  
```c
#include <pthread.h>

int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
```
以读模式加锁，`tryrdlock`是非阻塞版本。

#### pthread_rwlock_wrlock / pthread_rwlock_trywrlock
函数原型：  
```c
#include <pthread.h>

int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
```
以写模式加锁，`trywrlock`是非阻塞版本。

#### pthread_rwlock_unlock
函数原型：  
```c
#include <pthread.h>

int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```
解锁，无论读写模式加的锁，都使用该函数解锁。

### 练习 - 使用读写锁
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

int global_counter;
pthread_rwlock_t rwlock;

void* th_write(void* arg) {
    int i = (int)arg;
    int t;
    while (1) {
        pthread_rwlock_wrlock(&rwlock);
        t = global_counter;
        usleep(1000); // 抱着睡 浪费时间
        printf("==write(%d - tid %lu) : %d, ++ counter %d\n", i, pthread_self(), t, ++ global_counter);
        pthread_rwlock_unlock(&rwlock);
        usleep(10000);
    }
    return NULL;
}

void* th_read(void* arg) {
    int i = (int)arg;
    while (1) {
        pthread_rwlock_rdlock(&rwlock);
        printf("--read (%d - tid %lu) : %d\n", i, pthread_self(), global_counter);
        pthread_rwlock_unlock(&rwlock);
        usleep(2000);
    }
    return NULL;
}

int main(int argc, char** argv) {
    // 错误码判断就不写了 ^^
    int ret;
    pthread_t tids[8];
    ret = pthread_rwlock_init(&rwlock, NULL);
    for (int i = 0; i < 3; ++ i) { // 3 write thread
        ret = pthread_create(tids + i, NULL, th_write, (void*)i);
    }
    for (int i = 0; i < 5; ++ i) { // 5 read  thread
        ret = pthread_create(tids + i + 3, NULL, th_read, (void*)i);
    }
    for (int i = 0; i < 8; ++ i) {
        ret = pthread_join(tids[i], NULL);
    }
    ret = pthread_rwlock_destroy(&rwlock);
    return 0;
}
```

## 条件变量
条件变量，**不是锁**，但它可以阻塞线程。和互斥锁配合使用。给多线程提供一个会合的场所。

### 条件变量的操作函数
主要有这几个：  
- `pthread_cond_init`：创建条件变量；
- `pthread_cond_destroy`：销毁条件变量；
- `pthread_cond_wait`：阻塞等待条件满足；
- `pthread_cond_timedwait`：限时等待条件变量；
- `pthread_cond_signal`：通知；
- `pthread_cond_broadcast`：广播；
- 返回值，成功都返回`0`，失败都返回错误号。

#### pthread_cond_init / pthread_cond_destroy
函数原型：  
```c
int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_init(pthread_cond_t *restrict cond,
    const pthread_condattr_t *restrict attr);
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```
`attr`：条件变量的属性，默认传`NULL`。

动态初始化`pthread_cond_init(&cond, NULL)`，静态初始化`pthread_cond_t cond = PTHREAD_COND_INITIALIZER`。

#### pthread_cond_wait
函数原型：  
```c
#include <pthread.h>

int pthread_cond_wait(pthread_cond_t *restrict cond,
    pthread_mutex_t *restrict mutex);
```
需要一个条件变量和一个互斥锁。

函数的作用：
1. **阻塞**等待条件变量`cond`满足；
2. 释放已掌握的互斥锁（解锁），相当于`pthread_mutex_unlock(&mutex)`（之前要加锁）
   - 1，2两步为一个**原子操作**
3. 当被唤醒（条件变量满足了），`pthread_cond_wait`函数返回时，解除阻塞并重新申请获得互斥锁（加锁）`pthread_mutex_lock(&mutex)`

条件满足的情况通过`pthread_cond_signal`和`pthread_cond_broadcast`通知。

![](/post_images/posts/Coding/【Linux系统编程】12/wait示意图.jpg "wait示意图")

#### pthread_cond_timedwait
函数原型：  
```c
#include <pthread.h>

int pthread_cond_timedwait(pthread_cond_t *restrict cond,
    pthread_mutex_t *restrict mutex,
    const struct timespec *restrict abstime);
```
- `abstime`：`struct timespec`类型的绝对时间的描述。

`man sem_timedwait`，查看`struct_timespec`：  
```c
struct timespec {
    time_t tv_sec;      /* Seconds */
    long   tv_nsec;     /* Nanoseconds [0 .. 999999999] */
};
```
该结构体表述的时间是**绝对时间**（从1970.01.01-00:00:01开始），正确用法如下：参阅APUE.11.6线程同步条件变量小节  
```c
time_t cur = time(NULL); // 获得当前时间
struct timespec t;
t.tv_sec = cur + 1; // 定时一秒
pthread_cond_timedwait(&cond, &mutex, &t);
```

#### pthread_cond_signal / pthread_cond_broadcast
函数原型：  
```c
#include <pthread.h>

int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_signal(pthread_cond_t *cond);
```
- signal：唤醒**至少一个**阻塞在该条件变量上的线程；（不绝对是一个）
- broadcast：唤醒全部阻塞在该条件变量上的线程。

### 生产者-消费者 条件变量模型
线程同步的经典案例即为生产者-消费者模型，借助条件变量来实现这一模型，是比较常见的一种方法。假定有两个线程，一个模拟生产者行为，一个模拟消费者行为。两个线程同时操作一个共享资源（一般称之为汇聚），生产者向其中添加产品，消费者从中消费掉产品。

![](/post_images/posts/Coding/【Linux系统编程】12/生产者消费者条件变量模型.jpg "生产者消费者条件变量模型")

示意图可能不太能理解，参考下面的实现代码。

### 练习1 - 生产者-消费者 一对一
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

// 共享数据 链表形式
struct msg{
    int num;
    struct msg* next;
};
// 初始链表中没有节点
struct msg* head;

// 静态初始化条件变量和互斥锁
pthread_cond_t cond = PTHREAD_COND_INITIALIZER; // 链表中是否新增数据了
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void err_pthread(int ret, char* str) {
    if (ret != 0) {
        fprintf(stderr, "%s : %s\n", str, strerror(ret));
        pthread_exit(NULL);
    }
}

void* procuder_pthread(void* arg) {
    while (1) {
        // 生产一个节点
        struct msg* m = malloc(sizeof(struct msg));
        m -> num = rand() % 1000 + 1;
        printf("++producer : %d\n", m -> num);
        // 将生产的节点放入共享链表
        int ret = pthread_mutex_lock(&mutex); // 尝试加锁
        err_pthread(ret, "pthread_mutex_lock error");
        m -> next = head; // 新节点头插 放入链表
        head = m;
        ret = pthread_mutex_unlock(&mutex); // 解锁
        err_pthread(ret, "pthread_mutex_unlock error");
        // 通知阻塞的消费者 有新的节点
        ret = pthread_cond_signal(&cond);
        err_pthread(ret, "pthread_cond_signal error");
        sleep(rand() % 3); // 放弃CPU
    }
    return NULL;
}

void* consumer_pthread(void* arg) {
    while (1) {
        struct msg* m;
        int ret = pthread_mutex_lock(&mutex); // 尝试加锁
        err_pthread(ret, "pthread_mutex_lock error");
        if (head == NULL) { // 拿到锁 但是没有数据 就阻塞等待条件变量通知 解锁 条件变量满足时再次尝试加锁
            ret = pthread_cond_wait(&cond, &mutex);
            err_pthread(ret, "pthread_cond_wait error");
        }
        // 跳出循环 1. 链表中有数据 2. 无数据 wait阻塞 条件变量满足通知 重新竞争到锁
        // 进行消费
        m = head; // 把链表的第一个节点取下
        head = m -> next;
        ret = pthread_mutex_unlock(&mutex); // 消费完共享数据 解锁
        err_pthread(ret, "pthread_mutex_unlock error");
        printf("--consumer : %d\n", m -> num); // 打印获得的节点
        free(m); // 释放内存
        sleep(rand() % 3); // 放弃CPU
    }
    return NULL;
}

int main(int argc, char** argv) {
    int ret;
    srand(time(NULL));
    pthread_t prod_tid, cons_tid;
    // 创建线程 生产者-消费者
    ret = pthread_create(&prod_tid, NULL, procuder_pthread, NULL);
    err_pthread(ret, "pthread_create error");
    ret = pthread_create(&cons_tid, NULL, consumer_pthread, NULL);
    err_pthread(ret, "pthread_create error");
    // 回收线程
    ret = pthread_join(prod_tid, NULL);
    err_pthread(ret, "pthread_join error");
    ret = pthread_join(cons_tid, NULL);
    err_pthread(ret, "pthread_join error");
    
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ ./cond_prod_cons 
++producer : 89
--consumer : 89
++producer : 820
--consumer : 820
++producer : 271
--consumer : 271
++producer : 725
--consumer : 725
++producer : 339
--consumer : 339
++producer : 841
++producer : 520
++producer : 64
--consumer : 64
--consumer : 520
--consumer : 841
++producer : 146
++producer : 771
--consumer : 771
++producer : 63
--consumer : 63
++producer : 591
--consumer : 591
--consumer : 146
^C
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ 
```

### 练习2 - 生产者-消费者 一对多
代码：  
```c
// 其他略了

void* consumer_pthread(void* arg) {
    while (1) {
        struct msg* m;
        int ret = pthread_mutex_lock(&mutex); // 尝试加锁
        err_pthread(ret, "pthread_mutex_lock error");
        // *******************************************
        while (head == NULL) { // 拿到锁 但是没有数据 就阻塞等待条件变量通知 解锁 条件变量满足时再次尝试加锁
            ret = pthread_cond_wait(&cond, &mutex);
            err_pthread(ret, "pthread_cond_wait error");
            // 当线程结束等待并获得锁时
            // 有可能共享内容被消费完了
            // 不能退出循环进行消费 需要重新等待生产者的通知
        }
        // ********************************************
        // 跳出循环 1. 链表中有数据 2. 无数据 wait阻塞 条件变量满足通知 重新竞争到锁
        // 进行消费
        m = head; // 把链表的第一个节点取下
        head = m -> next;
        ret = pthread_mutex_unlock(&mutex); // 消费完共享数据 解锁
        err_pthread(ret, "pthread_mutex_unlock error");
        printf("--consumer(%lu) : %d\n", pthread_self(), m -> num); // 打印获得的节点
        free(m); // 释放内存
        sleep(rand() % 3); // 放弃CPU
    }
    return NULL;
}

int main(int argc, char** argv) {
    int ret;
    srand(time(NULL));
    pthread_t prod_tid, cons_tids[5];
    // 创建线程 生产者-消费者
    ret = pthread_create(&prod_tid, NULL, procuder_pthread, NULL);
    err_pthread(ret, "pthread_create error");
    for (int i = 0; i < 5; ++ i) {
        ret = pthread_create(cons_tids + i, NULL, consumer_pthread, NULL);
        err_pthread(ret, "pthread_create error");
    }
    // 回收线程
    ret = pthread_join(prod_tid, NULL);
    err_pthread(ret, "pthread_join error");
    for (int i = 0; i < 5; ++ i) {
        ret = pthread_join(cons_tids[i], NULL);
        err_pthread(ret, "pthread_join error");
    }
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ ./cond_prod_cons_more 
++producer : 794
--consumer(140171192317696) : 794
++producer : 480
--consumer(140171183924992) : 480
++producer : 375
--consumer(140171089540864) : 375
++producer : 910
--consumer(140171175532288) : 910
++producer : 50
--consumer(140171167139584) : 50
^C
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ 
```

### 条件变量的优点
相较于 mutex 而言，条件变量可以减少竞争。 

如直接使用 mutex，除了生产者、消费者之间要竞争互斥量以外，消费者之间也需要竞争互斥量，但如果汇聚（链表）中没有数据，消费者之间竞争互斥锁是无意义的。有了条件变量机制以后，只有生产者完成生产，才会引 起消费者之间的竞争。提高了程序效率。 

## 信号量 semaphore
可以看作是进阶版的互斥量（初始化值为N的互斥量）。N表示可以同时访问共享数据区的线程数。

由于互斥锁的粒度比较大，如果我们希望在多个线程间对某一对象的部分数据进行共享，使用互斥锁是没有办法实现的，只能将整个数据对象锁住。这样虽然达到了多线程操作共享数据时保证数据正确性的目的，却无形中导致线程的并发性下降。线程从并行执行，变成了串行执行。与直接使用单进程无异。 

信号量，是相对折中的一种处理方式，既能保证同步，数据不混乱，又能提高线程并发。 

### 信号量的基本操作
- `sem_wait`：类似于`pthread_mutex_lock`
  - 如果信号量大于0，则将信号量减一；
  - 如果信号量为0，线程阻塞。
- `sem_post`：类似于`phtread_mutex_unlock`
  - 将信号量加一，同时唤醒阻塞在信号量上的线程。
- `sem_t`的实现对用户是隐藏的，所以对信号量的加减操作只能通过函数完成；
- 信号量的初值，决定了能占用信号量的线程的个数。

### 信号量操作函数
主要操作函数如下：
- `sem_init`：创建信号量；
- `sem_destroy`：销毁信号量；
- `sem_wait`：相当于mutex加锁；
- `sem_trywait`：非阻塞的wait；
- `sem_timedwait`：有限时的wait；
- `sem_post`：相当于mutex解锁；
- 它们的返回值，成功返回`0`，失败返回`-1`，并设置`errno`。

注意，信号量相关函数没有`pthread`前缀，错误返回方式也和`pthread_*`不同。

（信号量不仅可以使用在线程中，也可以使用在进程中）

#### sem_init / sem_destroy
函数原型：  
```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);      int sem_destroy(sem_t *sem);
```
- `pshared`：process shared，表示是否在进程间共享
  - `0`：用于线程间同步；
  - 非零，通常为`1`：用于进程间同步。
- `value`：指定可以同时访问的线程数。

好像不能静态初始化。

#### sem_wait / sem_post
函数原型：  
```c
#include <semaphore.h>

int sem_wait(sem_t *sem); // -- 
int sem_post(sem_t *sem); // ++
```

#### sem_trywait / sem_timedwait
函数原型：  
```c
#include <semaphore.h>

int sem_trywait(sem_t *sem);
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);

struct timespec {
    time_t tv_sec;      /* Seconds */
    long   tv_nsec;     /* Nanoseconds [0 .. 999999999] */
};
```
- trywait：非阻塞的wait；
- timedwait：有限时的wait；
- `abs_timeout`，使用的是绝对时间（从1970.01.01-00:00:01开始）

正确使用方法：  
```c
tiem_t cur = time(NULL);
struct timespec t;
t.tv_sec = cur + 1; // 定时一秒
sem_timedwait(&sem, &t);
```

### 生产者-消费者 信号量模型

![](/post_images/posts/Coding/【Linux系统编程】12/生产者消费者信号量模型.jpg "生产者消费者信号量模型")

### 生产者-消费者 一对一实现
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <semaphore.h>

// 错误输出就不写了^^

#define NUM 5
// 存放产品的循环队列
int queue[NUM];
// 空格子信号量  产品数信号量
sem_t blank_num, product_num;

void* producer(void* arg) {
    int i = 0;
    while (1) {
        sem_wait(&blank_num); // --空格子数
        queue[i] = rand() % 1000 + 1; // 生产一个产品
        printf("++produce : %d\n", queue[i]);
        sem_post(&product_num); // ++ 产品数
        i = (i + 1) % NUM; // 循环队列
        sleep(rand() % 1); // CPU让给消费者
    }
    pthread_exit(NULL);
}

void* consumer(void* arg) {
    int i = 0;
    while (1) {
        sem_wait(&product_num); // -- 产品数
        printf("--consumer : %d\n", queue[i]); // 消费一个商品
        queue[i] = 0;
        sem_post(&blank_num); // ++ 格子数
        i = (i + 1) % NUM;
        sleep(rand() % 3); // 让出CPU
    }
    pthread_exit(NULL);
}

int main(int argc, char** argv) {
    pthread_t prod_tid, cons_tid;
    sem_init(&blank_num, 0, NUM); // 初始空格子5个
    sem_init(&product_num, 0, 0); // 初始产品0个
    pthread_create(&prod_tid, NULL, producer, NULL);
    pthread_create(&cons_tid, NULL, consumer, NULL);
    pthread_join(prod_tid, NULL);
    pthread_join(cons_tid, NULL);
    sem_destroy(&blank_num);
    sem_destroy(&product_num);
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ ./sem_prod_cons 
++produce : 384
--consumer : 384
++produce : 916
++produce : 336
--consumer : 916
++produce : 650
++produce : 363
++produce : 691
--consumer : 336
++produce : 927
++produce : 427
--consumer : 650
++produce : 212
--consumer : 363
++produce : 430
--consumer : 691
++produce : 863
^C
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thdsyn$ 
```
