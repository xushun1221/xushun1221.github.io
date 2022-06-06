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
```c
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

动态创建读写锁`pthread_rwlock_init(rwlock, NULL)`，静态创建读写锁`pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER`，效果相同。

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
条件变量，不是锁，但它可以阻塞线程。和互斥锁配合使用。给多线程提供一个会合的场所。

### 条件变量的操作函数
主要有这几个：  
- `pthread_cond_init`
- `pthread_cond_destroy`
- `pthread_cond_wait`
- `pthread_cond_timedwait`
- `pthread_cond_signal`
- `pthread_cond_broadcast`









## 信号量
