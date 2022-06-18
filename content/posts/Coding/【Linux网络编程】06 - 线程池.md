---
title: 【Linux网络编程】06 - 线程池
date: 2022-06-17
tags: [Linux, network]
categories: [Coding]
---

## 线程池

### 为什么要使用线程池？
回顾之前写的多线程服务器，主线程使用监听套接字循环accept客户端连接，一旦有新的客户端连接，就创建一个新的线程去处理这个客户端发来的数据，当客户端关闭时，再销毁这个线程。使用这种方式提供服务，需要不断地创建销毁线程，且CPU需要再多个线程之间来回切换，系统资源消耗比较大。相比之下，多路IO转接服务器只需要一个线程（进程）就可以实现，开销较小，所以效率比较高。

由于创建和销毁线程的开销比较大，所以不在服务运行过程中一个个地创建销毁线程，而是在服务启动之前批量创建多个线程，并将它们统一管理，这就是**线程池**的概念。当server中有任务需要处理时，唤醒一个线程池中的线程对其进行处理，处理完成后不销毁线程，线程重新回到线程池中等待再次被唤醒。

之前写的多线程、多路IO转接等服务器主要实现的是，客户端如何和客户端建立连接接收请求，而线程池主要处理的是，服务器拿到客户请求后，该如何处理请求的部分。（可以将多路IO转接和线程池结合起来使用）

### 线程池实现逻辑
`main`  
1. 创建线程池
2. 向线程池中添加任务，借助回调函数处理
3. 销毁线程池

`threadpool_create`  
1. 创建一个线程池结构体指针并分配内存
2. 初始化线程池相关描述信息
3. 为线程数组分配空间
4. 为任务队列分配空间
5. 初始化互斥锁和条件变量
6. 启动最小数量的任务处理线程
7. 启动线程池管理线程
8. 如果某一步失败，回收所有分配的空间

`threadpool_destroy`  
1. 关闭线程池
2. 销毁管理线程
3. 通知所有空闲线程销毁
4. 回收剩下的线程
5. 销毁线程池结构体

`threadpool_adjust`(管理线程回调函数)  
1. 每过一定时间，对线程池进行管理（循环）
2. 对线程池结构体加锁并获取相关的线程池信息
3. 判断是否需要创建更多的线程，如果需要添加一批新线程
4. 判断是否需要销毁多余的空闲线程，如果需要销毁一批线程（向线程发送通知，让它们自行结束）

`threadpool_thread`(任务处理线程回调函数)  
1. 循环等待任务
2. 对线程池加锁
3. 如果任务队列为空，阻塞等待唤醒，再次竞争加锁
4. 如果被唤醒时，需要销毁的线程数量大于0，结束当前的线程
5. 如果线程池关闭了，结束当前线程
6. 正常执行任务的情况
7. 从任务队列中取出一个任务
8. 通知可以有新的任务加入
9. 立即释放线程池
10. 处理任务

`threadpool_add`  
1. 对线程池加锁
2. 如果任务队列已满，阻塞等待通知
3. 如果线程池已经关闭，通知所有空闲线程
4. 正常添加任务
5. 将任务添加到任务队列队尾
6. 唤醒线程池中等待处理任务的线程

### 实现
```c
/*
@Filename : thread_pool.c
@Description : implementation of thread pool 
@Datatime : 2022/06/17 09:08:05
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <signal.h>

#define DEFAULT_TIME 10         // 管理线程每10秒检测一次
#define MIN_WAIT_TASK_NUM 10    // 如果 queue_size >= MIN_WATI_TASK_NUM 添加新的线程到线程池
#define DEFAULT_THREAD_VARY 10  // 每次创建和销毁的线程的个数
#define true 1
#define false 0

// 线程池对应的任务
typedef struct {
    void* (*function)(void* arg);
    void* arg;
} threadpool_task_t;

// 线程池
typedef struct {
    pthread_mutex_t lock;           // 锁住本结构体
    pthread_mutex_t thread_counter; // 记录忙状态的线程个数 busy_thread_num

    pthread_cond_t queue_not_full;  // 任务队列满 添加任务的线程阻塞
    pthread_cond_t queue_not_empty; // 任务队列空 处理任务的线程阻塞

    pthread_t* threads;             // 线程池数组 存放每个线程的tid
    pthread_t adjust_thread;        // 管理线程的tid
    threadpool_task_t* task_queue;  // 任务队列首地址 (循环队列)

    int min_thread_num;       // 线程池最小线程数
    int max_thread_num;       // 线程池最大线程数
    int live_thread_num;      // 当前存活的线程数
    int busy_thread_num;      // 当前忙的线程数
    int wait_exit_thread_num; // 要销毁的线程数

    int queue_front;    // 队头下标
    int queue_rear;     // 队尾下标
    int queue_size;     // 任务队列实际任务数
    int queue_max_size; // 任务队列最大任务数

    int shutdown; // 线程池使用状态
} threadpool_t;


void* threadpool_thread(void* threadpool);
int threadpool_add(threadpool_t* threadpool, void*(*function)(void* arg), void* arg);
void* process(void* arg);
threadpool_t* threadpool_create(int min_thread_num, int max_thread_num, int queue_max_size);
void* threadpool_adjust(void* threadpool);
int is_thread_alive(pthread_t tid);
int threadpool_destroy(threadpool_t* threadpool);
int threadpool_free(threadpool_t* threadpool);



int threadpool_free(threadpool_t* threadpool) {
    threadpool_t* pool = threadpool;
    if (pool == NULL) {
        return -1;
    }
    if (pool -> task_queue) {
        free(pool -> task_queue);
    }
    if (pool -> threads) {
        free(pool -> threads);
        pthread_mutex_lock(&(pool -> lock));
        pthread_mutex_destroy(&(pool -> lock));
        pthread_mutex_lock(&(pool -> thread_counter));
        pthread_mutex_destroy(&(pool -> thread_counter));
        pthread_cond_destroy(&(pool -> queue_not_empty));
        pthread_cond_destroy(&(pool -> queue_not_full));
    }
    free(pool);
    pool = NULL;
    return 0;
}

int threadpool_destroy(threadpool_t* threadpool) {
    threadpool_t* pool = threadpool;
    if (pool == NULL) {
        return -1;
    }
    pool -> shutdown = true;
    // 先销毁管理线程
    pthread_join(pool -> adjust_thread, NULL);
    // 通知所有的空闲线程销毁
    for (int i = 0; i < pool -> live_thread_num; ++ i) {
        pthread_cond_broadcast(&(pool -> queue_not_empty));
    }
    // 回收剩下的线程
    for (int i = 0; i < pool -> live_thread_num; ++ i) {
        pthread_join(pool -> threads[i], NULL);
    }
    threadpool_free(pool);
    return 0;
}

int is_thread_alive(pthread_t tid) {
    // 发0号信号，测试线程是否存活
    int kill_rc = pthread_kill(tid, 0);     
    if (kill_rc == ESRCH) {
        return false;
    }
    return true;
}

void* threadpool_adjust(void* threadpool) {
    threadpool_t* pool = (threadpool_t*) threadpool;
    while (pool -> shutdown == 0) {
        // 定时对线程池进行管理
        sleep(DEFAULT_TIME);
        // 根据 queue_size live_thread_num busy_thread_num 确定是否需要创建或销毁线程
        pthread_mutex_lock(&(pool -> lock));
        int queue_size = pool -> queue_size;
        int live_thread_num = pool -> live_thread_num;
        pthread_mutex_unlock(&(pool -> lock));
        pthread_mutex_lock(&(pool -> thread_counter));
        int busy_thread_num = pool -> busy_thread_num;
        pthread_mutex_unlock(&(pool -> thread_counter));
        // 如果需要 创建更多的新线程
        if (queue_size >= MIN_WAIT_TASK_NUM && live_thread_num < pool -> max_thread_num) {
            pthread_mutex_lock(&(pool -> lock));
            // 遍历pool -> threads 将线程添加到空的里面
            // 一次增加DEFAULT_THREAD_VARY个线程
            int add = 0;
            for (int i = 0; i < pool -> max_thread_num && add < DEFAULT_THREAD_VARY && pool -> live_thread_num < pool -> max_thread_num; ++ i) {
                if (pool -> threads[i] == 0 || is_thread_alive(pool -> threads[i]) == false) {
                    pthread_create(&(pool -> threads[i]), NULL, threadpool_thread, (void*)pool);
                    add ++;
                    pool -> live_thread_num ++;
                }
            }
            pthread_mutex_unlock(&(pool -> lock));
        }
        // 如果需要 销毁多余的空闲进程
        if ((busy_thread_num * 2) < live_thread_num && live_thread_num > pool -> min_thread_num) {
            // 一次销毁DEFAULT_THREAD_VARY个 随即即可
            pthread_mutex_lock(&(pool -> lock));
            pool -> wait_exit_thread_num = DEFAULT_THREAD_VARY;
            pthread_mutex_unlock(&(pool -> lock));
            // 通知处在空闲状态的进程 它们会自行终止
            for (int i = 0; i < DEFAULT_THREAD_VARY; ++ i) {
                pthread_cond_signal(&(pool -> queue_not_empty));
            }
        }
    }
    pthread_exit(NULL);
}

void* threadpool_thread(void* threadpool) {
    threadpool_t* pool = (threadpool_t*)threadpool;
    threadpool_task_t task;
    while (true) {
        // 尝试对线程池加锁
        pthread_mutex_lock(&(pool -> lock));
        // 如果任务队列为空 则解锁阻塞等待唤醒 再次竞争加锁
        // 如果有任务 则跳过该while
        while ((pool -> queue_size == 0) && (pool -> shutdown == false)) {
            printf("thread 0x%x is waiting\n", (unsigned)pthread_self());
            pthread_cond_wait(&(pool -> queue_not_empty), &(pool -> lock));
            // 如果要销毁的线程数量大于0 则销毁当前的线程
            // 此时被唤醒就是为了销毁多余的线程
            if (pool -> wait_exit_thread_num > 0) {
                pool -> wait_exit_thread_num --;
                // 如果线程池里线程个数大于最小值时 可以结束当前进程
                if (pool -> live_thread_num > pool -> min_thread_num) {
                    printf("thread 0x%x is exiting\n", (unsigned)pthread_self());
                    pool -> live_thread_num --;
                    pthread_mutex_unlock(&(pool -> lock));
                    pthread_exit(NULL);
                }
            }
        }
        // 如果线程池关闭了 自行退出
        if (pool -> shutdown == true) {
            pthread_mutex_unlock(&(pool -> lock));
            printf("thread 0x%x is exiting\n", (unsigned)pthread_self());
            pthread_detach(pthread_self()); // 清理
            pthread_exit(NULL);
        }
        // 从任务队列中取出一个任务
        task.function = pool -> task_queue[pool -> queue_front].function;
        task.arg = pool -> task_queue[pool -> queue_front].arg;
        pool -> queue_front = (pool -> queue_front + 1) % pool -> queue_max_size;
        pool -> queue_size --;
        // 通知可以有新的任务加入进来
        pthread_cond_broadcast(&(pool -> queue_not_full));
        // 立即释放lock
        pthread_mutex_unlock(&(pool -> lock));
        // 处理任务
        printf("thread 0x%x start working\n", (unsigned)pthread_self());
        pthread_mutex_lock(&(pool -> thread_counter));
        pool -> busy_thread_num ++;
        pthread_mutex_unlock(&(pool -> thread_counter));
        (*(task.function))(task.arg);
        // 处理任务完成
        printf("thread 0x%x end working\n", (unsigned)pthread_self());
        pthread_mutex_lock(&(pool -> thread_counter));
        pool -> busy_thread_num --;
        pthread_mutex_unlock(&(pool -> thread_counter));
    }
    pthread_exit(NULL);
}


threadpool_t* threadpool_create(int min_thread_num, int max_thread_num, int queue_max_size) {
    threadpool_t* pool = NULL;
    do {
        // 线程池结构体
        pool = (threadpool_t*)malloc(sizeof(threadpool_t));
        if (pool == NULL) {
            printf("malloc threadpool_t fail\n");
            break;
        }
        // 初始化相关描述信息
        pool -> min_thread_num = min_thread_num;
        pool -> max_thread_num = max_thread_num;
        pool -> busy_thread_num = 0;
        pool -> live_thread_num = min_thread_num;
        pool -> wait_exit_thread_num = 0;
        pool -> queue_size = 0;
        pool -> queue_max_size = queue_max_size;
        pool -> queue_front = 0;
        pool -> queue_rear = 0;
        pool -> shutdown = 0;
        // 线程数组分配空间
        pool -> threads = (pthread_t*)malloc(sizeof(pthread_t) * max_thread_num);
        if (pool -> threads == NULL) {
            printf("malloc threads fail\n");
            break;
        }
        memset(pool -> threads, 0, sizeof(pthread_t) * max_thread_num);
        // 任务队列分配空间    
        pool -> task_queue = (threadpool_task_t*)malloc(sizeof(threadpool_task_t) * queue_max_size);
        if (pool -> task_queue == NULL) {
            printf("malloc task_queue fail\n");
            break;
        }
        // 初始化互斥锁和条件变量
        if (   pthread_mutex_init(&(pool -> lock), NULL) != 0
            || pthread_mutex_init(&(pool -> thread_counter), NULL) != 0
            || pthread_cond_init(&(pool -> queue_not_empty), NULL) != 0
            || pthread_cond_init(&(pool -> queue_not_full), NULL) != 0
        ) {
            printf("init lock or cond fail\n");
            break;
        }
        // 启动min_thread_num数量的线程
        int ret = 0;
        for (int i = 0; i < min_thread_num; ++ i) {
            // 创建任务处理线程 创建时将线程池结构体作为参数传递给线程
            ret = pthread_create(&(pool -> threads[i]), NULL, threadpool_thread, (void*)pool);
            if (ret == 0) {
                printf("create thread 0x%x...\n", (unsigned)(pool -> threads[i]));
            } else {
                printf("create pthread fail\n");
                break;
            }
        }
        if (ret != 0) {
            break;
        }
        // 创建管理者线程
        ret = pthread_create(&(pool -> adjust_thread), NULL, threadpool_adjust, (void*)pool);
        if (ret == 0) {
            printf("create thread(adjust) 0x%x...\n", (unsigned)(pool -> adjust_thread));
        } else {
            printf("create pthread fail\n");
            break;
        }
        // success create threadpool_t
        return pool;
    } while(0);
    // 如果前面的代码调用失败 释放分配的空间
    threadpool_free(pool);
    return NULL;
}

void* process(void* arg) {
    printf("thread 0x%x working on task %d\n", (unsigned)pthread_self(), *((int*)arg));
    // 模拟处理任务
    sleep(1);
    printf("task %d is end\n", *((int*)arg));
    return NULL;
}

int threadpool_add(threadpool_t* threadpool, void*(*function)(void* arg), void* arg) {
    threadpool_t* pool = threadpool;
    pthread_mutex_lock(&(pool -> lock));
    // 任务队列已满 阻塞
    while ((pool -> queue_size == pool -> queue_max_size) && (pool -> shutdown == false)) {
        pthread_cond_wait(&(pool -> queue_not_full), &(pool -> lock));
    }
    // 线程池已经关闭
    if (pool -> shutdown == true) {
        pthread_cond_broadcast(&(pool -> queue_not_empty));
        pthread_mutex_unlock(&(pool -> lock));
        return 0;
    }
    // 清空任务队列队尾元素的参数
    if (pool -> task_queue[pool -> queue_rear].arg != NULL) {
        pool -> task_queue[pool -> queue_rear].arg = NULL;
    }
    // 添加任务到队列里
    pool -> task_queue[pool -> queue_rear].function = function;
    pool -> task_queue[pool -> queue_rear].arg = arg;
    pool -> queue_rear = (pool -> queue_rear + 1) % pool -> queue_max_size;
    pool -> queue_size ++;
    // 添加完任务后唤醒线程池中等待处理任务的线程
    pthread_cond_signal(&(pool -> queue_not_empty));
    pthread_mutex_unlock(&(pool -> lock));
    return 0;
}


int main(int argc, char** argv) {
    // 创建线程池
    threadpool_t* pool;
    pool = threadpool_create(3, 100, 100);
    if (pool != NULL) {
        printf("thread pool created\n");
    } else {
        printf("thread pool create fail\n");
    }
    // 添加一些任务
    int nums[20];
    for (int i = 0; i < 20; ++ i) {
        nums[i] = i;
        printf("add task %d\n", i);
        // 向线程池中添加一个任务
        threadpool_add(pool, process, (void*)&nums[i]);
    }
    // 等待任务处理结束
    sleep(10);
    // 销毁线程池
    threadpool_destroy(pool);
    printf("thread pool destroyed\n");
    return 0;
}
```