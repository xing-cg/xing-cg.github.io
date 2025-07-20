---
title: Linux_线程
categories:
  - - Linux
  - - 操作系统
    - 多线程
tags: 
date: 2021/11/20
updated: 2025/7/20
comments: 
published:
---
# 与进程的区别

1. 进程：一个正在运行的程序，是资源分配的最小单位。
2. 线程：进程内部的一条执行路径，是执行任务的最小单位。

# 为什么需要线程

1. 程序需要同时做多个事情
2. 充分利用多处理器（多核）

# 示例
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
int my_global;
void * my_thread_handle(void * arg)
{
    int val;
    val = *((int*)arg);
    printf("new thread begin, got arg, val = %d\n", val);
    my_global += val;
    sleep(3);
    pthread_exit(&my_global);
    printf("new thread end\n");
}
int main()
{
    pthread_t mythread;
    int arg;
    void *thread_return_val;
    arg = 100;
    my_global = 1000;
    printf("my_global = %d\n", my_global);
    printf("ready create thread...\n");
    int error = pthread_create(&mythread, NULL, my_thread_handle, &arg);
    if (error)
    {
        printf("create thread failed!\n");
        exit(1);
    }
    printf("wait thread finished...\n");
    error = pthread_join(mythread, &thread_return_val);
    if (error)
    {
        printf("pthread_join failed!\n");
        exit(1);
    }
    printf("got return val, %d\n", *((int*)thread_return_val));
    printf("my_global = %d\n", my_global);
}
```
输出结果：
```
my_global = 1000
ready create thread...
wait thread finished...
new thread begin, got arg, val = 100
got return val, 1100
my_global = 1100
```
# API
需包含头文件`<pthread.h>`
编译链接时需带选项`-lpthread`
## `pthread_create`


```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
```


参数：
1. thread，指向新线程的标识符（需要传入已经定义好的`pthread_t`指针）
2. attr，用来设置新线程的属性。一般用默认属性NULL
3. `start_routine`，该线程的处理函数指针，返回类型、参数类型都是`void*`
4. arg，传给处理函数的参数。

返回值：
如果成功，将返回 0；如果出错，将返回一个错误号，并且`*thread`的内容未定义。
## `pthread_exit`

```c
void pthread_exit(void *retval);
```

在线程函数内部调用该函数，用于给外部写值。

## `pthread_join`
```c
int pthread_join(pthread_t thread, void **retval);
```
等待指定的线程结束。

参数：
1. thread，指定等待的线程
2. retval，指向该线程函数的返回值。线程函数的返回值类型是`void*`，所以该参数的类型为`void**`

如果成功，将返回 0；如果出错，将返回一个错误号


# 同步
安装posix手册：
```sh
sudo apt install manpages-posix-dev
```

C语言中，关于同步提供的四个方法：
1. 信号量
2. 互斥锁
3. 条件变量
4. 读写锁

信号量、互斥量用于解决多个线程对临界区的竞态问题。
如果只允许一个线程进入临界区则用互斥量；
如果要求多个线程的执行顺序满足某个约束，用信号量。
## 信号量
此时所指的“信号量”是指用于同一个进程内多个线程之间的信号量。
即POSIX信号量，面不是System V信号量（用于进程之间的同步）

用于线程的信号量的原理，与用于进程的信号量的原理相同。都有P、V操作。
### API
需要包含头文件`<semaphore.h>`

信号量的表示
```c
sem_t
```
信号量的初始化
```c
int sem_init(sem_t *sem, int pshared, unsigned int value);
```
参数：
1. sem，信号量的指针
2. pshared，0 表示该信号量不被其他进程共享；1 表示可被其他进程共享。
3. value，信号量的初值

成功时返回 0；在错误时返回 -1，并设置 errno 来指示错误。

---

信号量的P操作。想要取得访问权，对信号量减1。
```c
int sem_wait(sem_t *sem);
```
信号量的V操作。想要让渡访问权，对信号量加1。
```c
int sem_post(sem_t *sem);
```

---
信号量的析构
```c
int sem_destroy(sem_t *sem);
```
### 示例
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <semaphore.h>
#include <string.h>
#define BUFF_SIZE 80
char buff[BUFF_SIZE];
sem_t sem;
static void * str_thread_handle(void* arg)
{
    while(1)
    {
        if (sem_wait(&sem) != 0)
        {
            printf("sem wait failed!\n");
            exit(1);
        }
        printf ("string is: %s, len = %d\n", buff, strlen(buff));
    }
}
int main()
{
    int error;
    pthread_t str_thread;
    void * thread_return_val;

    error = sem_init(&sem, 0, 0);
    if (error != 0)
    {
        printf("sem init failed!\n");
        exit(1);
    }
    error = pthread_create(&str_thread, NULL, str_thread_handle, 0);
    if (error != 0)
    {
        printf("pthread create failed!\n");
        exit(1);
    }
    while (1)
    {
        fgets(buff, sizeof(buff), stdin);
        error = sem_post(&sem);
        if (error != 0)
        {
            printf("sem post failed!\n");
            exit(1);
        }
        if (strncmp(buff, "end", 3) == 0)
        {
            break;
        }
    }
    error = pthread_join(str_thread, &thread_return_val);
    if (error != 0)
    {
        printf("pthread join failed!\n");
        exit(1);
    }
    error = sem_destroy(&sem);
    if (error != 0)
    {
        printf("sem_destroy failed!\n");
        exit(1);
    }
}
```
结果
```
123（用户输入）
string is: 123
, len = 4
3（用户输入）
string is: 3
, len = 2
```

## 互斥量
效果上等同于初值为1的信号量。

（UNIX环境高级编程 第3版 中文版）为了避免多个线程同时对某公共数据操作时产生冲突，可以使用pthread的互斥接口来保护数据，**确保同一时间只有一个线程访问数据**。

互斥量(mutex)本质上来说是一把锁，利用互斥量，在访问共享资源前对互斥量进行设置（加锁），在访问完成后释放（解锁）互斥量，即可达到线程同步。

对互斥量加锁后，其他线程再次对mutex进行lock操作时会被**阻塞**，直到锁被释放。

> 如果释放互斥量时有一个以上的线程阻塞，那么所有该锁上的阻塞线程都会变成**可运行状态**，只有第一个变为运行态的线程可以拿到锁进行加锁，其他线程会看到互斥量依然是锁着的，只能回去再次等待锁重新变为可用。在这种方式下，每次只有一个线程可以向前执行。

### API
```c
pthread_mutex_t
```

```c
#include<pthread.h>
/* 函数的返回值：若成功返回0；否则返回错误编号 */
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

关于restrict关键字，请移步C/C++语言分栏。

要用默认的属性初始化互斥量，只需把attr设为NULL。

---

```c
#include<pthread.h>
/* 函数的返回值：若成功返回0；否则返回错误编号 */
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

如果线程不希望被阻塞，可以使用`pthread_mutex_trylock`尝试对互斥量进行加锁。
如果调用`pthread_mutex_trylock`时互斥量处于未锁状态，那么将锁住互斥量，返回0；否则就不能锁住互斥量，返回`EBUSY`。

### 避免死锁

如果线程试图对同一个互斥量加锁两次（拿到锁之后再加锁），那么它自身就会陷入死锁状态。

```c
pthread_mutex_t lock;
void * fun(void *arg)
{
    pthread_mutex_lock(&lock);
    pthread_mutex_lock(&lock);	//死锁
    //... 
}
int main()
{
    int res = pthread_mutex_init(&lock, NULL);
    if(res != 0)// 0 is success, or failed.
    {
        exit(1);
    }
    // ... initialization mutex
    
    pthread_t tid1;
    res = pthread_create(&tid1, NULL, fun, NULL);
    if(res != 0)exit(1);
    
    res = pthread_join(tid1, NULL);	//第二个参数用于接收线程的返回值，不感兴趣可以设置为NULL
    if(res != 0)exit(1);
    
    return 0;
}
```

还有其他方式也能产生死锁，例如，程序中使用一个以上的互斥量时，如果线程1一直占有互斥量1，它要进行下一步的条件是占有互斥量2，而互斥量2一直被线程2占有，线程2下一步的条件是占用互斥量1。即**两个线程都在互相请求另一个线程拥有的资源**。

可以通过仔细控制互斥量加锁的顺序来避免死锁的发生，例如，假设需要对两个互斥量A和B同时加锁。

如果所有线程总是在对互斥量B加锁之前能够锁住互斥量A，那么使用这两个互斥量就不会产生死锁。类似地，如果所有的线程总是在锁住互斥量A之前能够锁住互斥量B，那么也不会发生死锁。

可能出现的死锁只会发生在一个线程试图锁住另一个线程以相反的顺序锁住的互斥量。

有时候，应用程序的结构使得对互斥量进行排序是很困难的。如果涉及了太多的锁和数据结构，可用的函数并不能把它转换成简单的层次，那么就需要采用另外的方法。在这种情况下，B可以先释放占有的锁（因为A可能需要B占有的锁），然后过一段时间再试。

## 条件变量

条件变量是线程可用的另一种同步机制。

**条件变量给多个线程提供了一个回合的场所**。

**条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定的条件发生**。

条件本身是由互斥量保护的。**线程在改变条件状态之前必须首先锁住互斥量**。其他线程在获得互斥量之前不会察觉到这种改变，因为互斥量必须在锁定以后才能计算条件。


---
### API
```c
pthread_cond_t
```

```c
#include<pthread.h>
/* 函数的返回值：若成功，返回0；否则，返回错误编号 */
int pthread_cond_init(pthread_cond_t *restrict cond,
                      const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
```

在使用条件变量之前，必须先对它进行初始化。

在释放条件变量底层的内存空间之前，可以使用`pthread_cond_destroy`函数对条件变量进行析构。

---

```c
#include<pthread.h>
/* 函数的返回值：若成功，返回0；否则，返回错误编号 */
int pthread_cond_wait(pthread_cond_t *restrict cond,
                      pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
                           pthread_mutex_t *restrict mutex,
                           const struct timespec *restrict tsptr);
```

我们使用`pthread_cond_wait`等待条件变量变为真。如果在给定的时间内条件不能满足，那么会返回一个错误码。

传递给`pthread_cond_wait`的**互斥量对条件进行保护**。调用者把锁住的互斥量传给函数，**函数然后自动把调用线程放到等待条件的线程列表**上，**对互斥量解锁**。这就**关闭了条件检查**和**线程进入休眠状态等待条件改变**这两个操作**之间的时间通道**，这样线程就不会错过条件的任何变化。`pthread_cond_wait`返回时，互斥量再次被锁住。

> pthread_cond_timedwait函数多了一个超时参数`tsptr`。超时值指定了我们愿意等待多长时间，它是通过`timespec`结构指定的。需要指定愿意等待多长时间，这个时间值是一个**绝对数**而不是相对数。例如，假设愿意等待3分钟。那么，并不是把3分钟转换成`timespec`结构，而是需要把当前时间加上3分钟再转换成timespec结构。

>如果超时到期时条件还是没有出现，`pthread_cond_timewait`将重新获取互斥量，然后返回错误`ETIMEDOUT`。

>从`pthread_cond_wait`或者`pthread_cond_timedwait`调用成功返回时，线程需要重新计算条件，因为另一个线程可能已经在运行并改变了条件。

> 可以使用`clock_gettime`函数获取timespec结构表示的当前时间。但是目前并不是所有的平台都支持这个函数，因此，也可以用另一个函数`gettimeofday`获取`timeval`结构表示的当前时间，然后把这个时间转换成`timespec`结构。

---

有两个函数可以用于通知线程条件已经满足。`pthread_cond_signal`函数至少能唤醒一个等待该条件的线程，而`pthread_cond_broadcast`函数则能唤醒等待该条件的所有线程。

> POSIX规范为了简化`thread_cond_signal`的实现，允许他在实现的时候唤醒一个以上的线程。

```c
#include<pthread.h>
/* 函数的返回值：若成功，返回0；否则，返回错误编号 */
int pthread_cond_signal(pthread_cond_t * cond);
int pthread_cond_broadcast(pthread_cond_t * cond);
```

在调用`pthread_cond_signal`或者`pthread_cond_broadcast`时，我们说这是在给线程或者条件发信号。必须注意，一定要在改变条件状态以后再给线程发信号。

条件是工作队列的状态。我们用互斥量保护条件，在while循环中判断条件。把消息放到工作队列时，需要占有互斥量，但在给等待线程发信号时，不需要占有互斥量（但最好还是占有，详见[《Cpp_线程库》一篇中的《注意事项》一节](Cpp_线程库.md#注意事项)。只要线程在调用`pthread_cond_signal`之前把消息从队列中拖出了，就可以在释放互斥量以后完成这部分工作。因为我们是在while循环中检查条件，所以不存在这样的问题：线程醒来，发现队列仍为空，然后返回继续等待。如果代码不能容忍这种竞争，就需要在给线程发信号的时候占有互斥量。

## 读写锁（共享互斥锁）

读写锁和互斥量类似，不过读写锁允许更高的并行性。互斥量要么是锁住状态，要么是未锁状态，而且一次只有一个线程可以对其加锁。

读写锁有3种状态：**读模式下加锁状态**，**写模式下加锁状态**，**不加锁状态**。**一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁**。写与写、读都会冲突，读与读不冲突。

>1. 当读写锁是写加锁状态时，在这个锁被解锁之前，所有试图对这个锁加锁的线程都会被阻塞；
>2. 当读写锁在读加锁状态时，所有试图以读模式对它进行加锁的线程都可以得到访问权，但是任何希望以写模式对此锁进行加锁的线程都会阻塞，直到所有的线程释放它们的读锁为止。
>3. 虽然各操作系统对读写锁的实现各不相同，但当读写锁处于读模式锁住的状态，而这时有一个线程试图以写模式获取锁时，读写锁通常会阻塞随后的读模式锁请求。这样可以避免读模式锁长期占用，而等待的写模式锁请求一直得不到满足。

读写锁也叫做共享互斥锁(shared-exclusive lock)。**当读模式锁住时，相当于是以共享模式锁住的。当写模式锁住时，是以互斥模式锁住的**。

### API

```c
#include<pthread.h>
/* 函数的返回值：若成功，返回0；否则，返回错误编号 */
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                        pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

在释放读写锁占用的内存之前，需要调用pthread\_rwlock\_destroy做清理工作。如果pthread\_rwlock\_init为读写锁分配了资源，pthread\_rwlock\_destroy将释放这些资源。

**如果在调用pthread\_rwlock\_destroy之前就释放了读写锁占用的内存空间，那么分配给这个锁的资源就会丢失**。（分配给这个锁的资源指什么？“丢失”的具体含义？）

>1. 如果你的pthread\_rwlock\_t资源是临时变量，就要保证在生存期之前destroy；
>2. pthread_rwlock_t * 这个指针指向的内存区域是malloc动态分配的，那就是destroy后，再free释放空间；
>3. 你的pthread_rwlock_t如果是个全局变量，那么生存期其实就不影响，程序退出前destroy。

---

```c
#include<pthread.h>
/* 函数的返回值：若成功，返回0；否则，返回错误编号 */
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

要在读模式下锁定读写锁，需要调用pthread\_rwlock\_rdlock。要在写模式下锁定读写锁，需要调用pthread\_rwlock\_wrlock。不管以何种方式锁住读写锁，都可以调用pthread\_rwlock\_unlock进行解锁。

# 生产者-消费者模型

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <time.h>

#define BUFFER_MAX 10
#define PRODUCER_NUM 2
#define CONSUMER_NUM 3

sem_t empty;
sem_t full;
pthread_mutex_t mutex;

int buff[BUFFER_MAX];
int in_index = 0;
int out_index = 0;
void *produce(void *arg)
{
    int index = (int)arg;
    for (int i = 0; i < 30; i++)
    {
        sem_wait(&empty);
        pthread_mutex_lock(&mutex);
        buff[in_index] = rand() % 100;
        printf("the producer %d wrote data %d,where %d\n", index, buff[in_index], in_index);
        in_index = (in_index + 1) % BUFFER_MAX;
        pthread_mutex_unlock(&mutex);
        sem_post(&full);

        sleep(2);
    }
}
void *consume(void *arg)
{
    int index = (int)arg;
    for (int i = 0; i < 20; i++)
    {
        sem_wait(&full);
        pthread_mutex_lock(&mutex);
        printf("the consumer %d read the data %d,from %d\n", index, buff[out_index], out_index);
        out_index = (out_index + 1) % BUFFER_MAX;
        pthread_mutex_unlock(&mutex);
        sem_post(&empty);
        sleep(1);
    }
}

int main()
{
    sem_init(&empty, 0, BUFFER_MAX);
    sem_init(&full, 0, 0);
    pthread_mutex_init(&mutex, NULL);

    srand((int)time(NULL));

    // create the producers
    pthread_t producer_id[PRODUCER_NUM];
    for (int i = 0; i < PRODUCER_NUM; i++)
    {
        pthread_create(&producer_id[i], NULL, produce, (void *)i);
    }
    pthread_t consumer_id[CONSUMER_NUM];
    for (int i = 0; i < CONSUMER_NUM; i++)
    {
        pthread_create(&consumer_id[i], NULL, consume, (void *)i);
    }
    for (int i = 0; i < PRODUCER_NUM; i++)
    {
        pthread_join(producer_id[i], NULL);
    }
    for (int i = 0; i < CONSUMER_NUM; i++)
    {
        pthread_join(consumer_id[i], NULL);
    }
    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);
}
```

结果：
```
the producer 1 wrote data 96,where 0
the producer 0 wrote data 73,where 1
the consumer 1 read the data 96,from 0
the consumer 0 read the data 73,from 1
the producer 0 wrote data 54,where 2
the producer 1 wrote data 63,where 3
the consumer 0 read the data 54,from 2
the consumer 2 read the data 63,from 3
the producer 1 wrote data 22,where 4
the producer 0 wrote data 60,where 5
the consumer 1 read the data 22,from 4
the consumer 0 read the data 60,from 5
the producer 0 wrote data 20,where 6
the producer 1 wrote data 47,where 7
the consumer 1 read the data 20,from 6
the consumer 2 read the data 47,from 7
the producer 1 wrote data 7,where 8
the producer 0 wrote data 46,where 9
the consumer 1 read the data 7,from 8
the consumer 0 read the data 46,from 9
the producer 0 wrote data 32,where 0
the producer 1 wrote data 96,where 1
the consumer 0 read the data 32,from 0
the consumer 2 read the data 96,from 1
the producer 1 wrote data 61,where 2
the producer 0 wrote data 35,where 3
the consumer 0 read the data 61,from 2
the consumer 1 read the data 35,from 3
the producer 0 wrote data 29,where 4
the producer 1 wrote data 54,where 5
the consumer 0 read the data 29,from 4
the consumer 2 read the data 54,from 5
the producer 1 wrote data 85,where 6
the producer 0 wrote data 20,where 7
the consumer 0 read the data 85,from 6
the consumer 1 read the data 20,from 7
the producer 0 wrote data 13,where 8
the producer 1 wrote data 71,where 9
the consumer 0 read the data 13,from 8
the consumer 2 read the data 71,from 9
the producer 0 wrote data 4,where 0
the producer 1 wrote data 94,where 1
the consumer 0 read the data 4,from 0
the consumer 1 read the data 94,from 1
the producer 1 wrote data 70,where 2
the producer 0 wrote data 99,where 3
the consumer 0 read the data 70,from 2
the consumer 2 read the data 99,from 3
...
```

# 实现线程的三种模式

## 纯粹的用户级线程

## 纯粹的内核级线程

## 组合

## 区别

1. 用户级
   1. 创建开销小，可以创建很多。
   2. 无法使用多个处理器
2. 内核级
   1. 创建开销大。
   2. 由内核直接管理
   3. 可以使用多个处理器

# 线程安全



# 查看线程信息的方法

```bash
ps -ef | grep main
ps -eLf | grep main #UID PID PPID LWP（线程ID） C NLWP（线程数目）
ps -L #显示线程id
```

# 多线程中的fork

执行fork后，整个进程会被复制，但是规定只启用fork所在的那条执行路径。

# Linux线程的优势

1. 外部获得线程的返回值？
2. 

