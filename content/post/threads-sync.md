---
title: "Threads Sync"
description: 
categories:
  - "Linux"
tags:
  - "Linux"
date: 2021-05-12T00:48:23-07:00
slug: "threads-sync"
math: 
license: 
hidden: false
draft: false
---

# 引言

由于每个进程有自己独立的虚拟地址空间，为了打破进程与进程之间的“柏林墙”而实现通信，多进程更多的是考虑进程之间如何通信的问题；

而同一进程内的多个线程共享同一地址空间，为了避免多个线程同时访问数据造成的混乱，多线程之间更多的是考虑线程之间的同步问题。

> 所谓同步，即协同步调，按预定的先后次序访问共享资源，以免造成混乱。

线程同步：即当有一个线程在对内存进行操作时，其他线程都不可以对这个内存地址进行操作，直到该线程完成操作， 其他线程才能对该内存地址进行操作，而其他线程又处于等待状态。

线程同步的实现方式有6种：互斥量、读写锁、条件变量、自旋锁、屏障、信号量。

> 由于笔者主要学习UNIX环境的编程，所以这里只介绍UNIX环境常用的线程同步方式，即POSIX线程库（Pthreads）提供的线程同步接口。本文不会涉及Windows端的线程同步方式，如临界区（CriticalSection）等。

# 生产者-消费者模型

为了更好地讲清楚以下的线程同步方式，我决定介绍每种线程同步方式时，都搭配一个实际的应用案例。这个应用案例我决定选择生产者-消费者模型这一经典问题。

问题描述：有一群**生产者进程**在生产产品，并将这些产品提供给**消费者进程**进行消费，生产者进程和消费者进程可以**并发执行**，在两者之间设置了一个具有**n个缓冲区**的缓冲池，生产者进程需要将所生产的产品放到一个缓冲区中，消费者进程可以从缓冲区中取走产品消费。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/productor-consumer-model.png)

当生产者生产了一个产品之后，缓冲区里的产品就会+1；同样，如果消费者从缓冲区里边消费一个产品，缓冲区里的产品就会-1。这看起来没有任何问题。

但是在计算机中，这个缓冲区是位于**高速缓存或主存**上的，如果说生产者或消费者要操作里边的数据时，就分为三个步骤：

- 取出数据放到寄存器中 **register = count**；

- 在CPU的寄存器中将**register = register±1**；register = register + 1表示生产者生产了一个产品；register = register - 1表示消费者消费了一个产品。

- 将register放回缓冲区 **count = register**。

当生产者和消费者**并发**执行的时候，这就会出现问题，因为上述三个操作不具备**原子性**，即生产者和消费者的这三个操作可能会交叉执行，而不是一方执行完，另一方再执行。见下图，红色部分为生产者生产的过程，蓝色的为消费者消费的过程：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/execute-mix.png)

图右描述了一种可能出现的执行流程（假设count初始值为10）：

1. 生产者线程获得时间片，执行其第一步：把count值放到寄存器，即register = count；执行之后生产者线程私有的register和共享的count值均为10；
2. 生产者线程时间片未耗尽，执行其第二步：在寄存器中加1，即register = register + 1；执行之后生产者线程私有的register值为11，共享的count值为10；
3. 此时，生产者线程时间片耗尽，CPU调度消费者进程执行，消费者执行其第一步：把count值放到寄存器，即register = count；执行之后消费者线程私有的register和共享的count值均为10；
4. 消费者时间片未耗尽，消费者执行其第二步：在寄存器中减1，即register = register - 1；执行之后消费者线程私有的register值为9，共享的count值为10；
5. 消费者时间片未耗尽，消费者执行其第三步：把register值放回缓冲区，即count = register；执行之后消费者私有的register和共享的count值均为9；
6. 此时，消费者线程时间片耗尽，CPU调度生产者进程执行，生产者继续执行它的第三步：count = register；执行之后生产者私有的register和共享的count值均为11。

问题就出现了：整个过程中生产者生产了一个产品，消费者消费了一个产品，那count值前后应该不变，仍为10才对，现在却是11，说明这个数据是错误的。错误的原因就在于这两个进程**并发的执行**，他们轮流在操作缓冲区，导致缓冲区中的数据不一致，这个就是生产者-消费者的问题。

以下程序模拟了生产者消费者问题：生产者生产1亿个产品，消费者消费1亿个产品，最终num应该为0。但事实并非如此。

```c
#include<stdio.h>
#include<pthread.h>

int num = 0;//临界资源
void *producer(void* arg){
        int times = 100000000;//循环一亿次
        while(times--)
                num += 1;//每次生产一个产品
}

void *consumer(void* arg){
        int times = 100000000;
        while(times--)
                num -= 1;//每次消费一个产品
}

int main(){
        pthread_t thread_prod,thread_cons;
        pthread_create(&thread_prod, NULL, &producer, NULL);
        pthread_create(&thread_cons, NULL, &consumer, NULL);
        pthread_join(thread_prod, NULL);
        pthread_join(thread_cons, NULL);
        printf("num = %d\n", num);
}
```

该程序每次执行的结果并不相同，num值不为0：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/code-of-pro-con.png)

> 关于这个程序有一点要注意，循环次数要设的大一点。我一开始设的100万次，由于100万次太少了，每个线程的时间片足够一次执行完所有循环而不被其他线程打断，所以最终会输出num等于0。
>
> 设置成1亿次之后就好了，线程的时间片无法一次执行完所有循环，执行完一部分就被CPU喊停，调度另一线程执行了，所以会出现num值不为0；且由于每次执行，线程之间的调度无法原样重现，所以num值每次都不一样。

# 互斥量

互斥量（mutex）本质上说是一把锁，在访问共享资源前对互斥量进行设置（加锁），在访问完成后释放（解锁）互斥量。

对互斥量加锁以后，任何其他试图再次对互斥量加锁的线程都会被阻塞，直至当前线程释放该互斥量。

## POSIX互斥量API

POSIX线程库（Pthreads）提供了互斥量接口，互斥量用pthread_mutex_t数据类型来表示。

使用互斥量前必须首先进行初始化，有两种初始化方式：

- 把它设置为常量PTHREAD_MUTEX_INITIALIZER（仅适用于静态分配的互斥量）；
- 调用pthread_mutex_init函数进行初始化。

对于动态分配的互斥量（如new和malloc），在释放内存前需要调用pthread_mutex_destroy函数。

其常见API如下：

```c
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
//返回值：成功返回0；出错返回错误编号
```

> 要用默认属性初始化互斥量，只需把attr参数设为NULL。

```c
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
//返回值：成功返回0；出错返回错误编号
```

- pthread_mutex_lock函数用于给互斥量加锁，如果互斥量已上锁，则调用该函数的线程将阻塞直到互斥量被解锁；
- 如果线程不希望被阻塞，它可以使用pthread_mutex_trylock函数尝试对互斥量进行加锁，如果未加锁，则锁住互斥量并返回0；如果已加锁，则返回EBUSY；
- pthread_mutex_unlock函数用于对互斥量解锁。

```c
#include <pthread.h>
#include <time.h>
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex,
                            const struct timespec *restrict tsptr);
//成功返回0；出错返回错误编号
```

- 当线程试图获取一个已加锁的互斥量时，pthread_mutex_timedlock函数允许绑定线程阻塞时间。在达到超时时间值时，该函数不会对互斥量加锁，而是返回错误码ETIMEDOUT。

> 超时指定愿意等待的绝对时间，用timespec结构表示，以秒和纳秒描述时间。

## 互斥量解决生产者消费者问题

程序如下：

```c
#include <stdio.h>
#include <pthread.h>

int num = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *producer(void* arg){
        int times = 100000000;
        while(times--){
                pthread_mutex_lock(&mutex);
                num += 1;
                pthread_mutex_unlock(&mutex);
        }
}

void *consumer(void* arg){
        int times = 100000000;
        while(times--){
                pthread_mutex_lock(&mutex);
                num -= 1;
                pthread_mutex_unlock(&mutex);
        }
}

int main(){
        pthread_t thread_prod, thread_cons;
        pthread_create(&thread_prod, NULL, &producer, NULL);
        pthread_create(&thread_cons, NULL, &consumer, NULL);
        pthread_join(thread_prod, NULL);
        pthread_join(thread_cons, NULL);
        printf("num = %d\n", num);
}
```

直接结果如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/mutex.png)

可见，使用互斥量同步多线程共享资源的访问后，输出就如预期了，每次输出都是0。

> 执行的时候会明显感觉到很慢，特意用time命令测试了下，上述程序需要近4秒钟才可以执行完。所以加锁会带来性能的损耗。

# 读写锁

互斥量只有两种状态：加锁和不加锁；且一次只有一个线程可以对其加锁；

读写锁有三种状态：读模式加锁、写模式加锁和不加锁；一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。

> 读写锁非常适合对数据结构读的次数远大于写的情况。

## POSIX读写锁API

读写锁在使用之前必须初始化，在释放它们的底层内存之前必须销毁。

```c
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
//两个函数返回值：成功返回0；出错返回错误编号
```

> 要用默认属性初始化读写锁，只需把attr参数设为NULL。
>
> 对于静态分配的读写锁，也可以使用常量PTHREAD_RWLOCK_INITIALIZER来初始化。

```c
#include <pthread.h>
int pthread_mutex_rdlock(pthread_rwlock_t *rwlock);
int pthread_mutex_wrlock(pthread_rwlock_t *rwlock);
int pthread_mutex_unlock(pthread_rwlock_t *rwlock);
//成功返回0；出错返回错误编号
```

- pthread_mutex_rdlock函数以读模式获取读写锁；
- pthread_mutex_wrlock函数以写模式获取读写锁；
- pthread_mutex_unlock函数用于解锁。

```c
#include <pthread.h>
#include <time.h>
int pthread_rwlock_timedrdlock(pthread_rwlock_t *restrict rwlock,
                               const struct timespec *restrict tsptr);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *restrict rwlock,
                               const struct timespec *restrict tsptr);
//成功返回0；出错返回错误编号
```

上边两个函数用于指定线程应该停止阻塞的时间。

## 读写锁解决读者写者问题

读者写者问题与之前的生产者消费者问题不同：读者线程只去读取共享资源但不会修改它，而写者线程会修改共享资源。用读写锁可以很好的解决读者写者问题。读者线程以读模式获取锁，这样不影响其他线程以读模式获取锁；写者线程以写模式获取锁，这样在修改共享资源期间，其他线程无法访问该共享资源。

> 生产者消费者问题中的生产者线程和消费者线程实际上都是“写者”线程，因为它们都会修改共享资源。

以下程序使用读写锁（互斥量）解决了读者写者问题。

```c
#include <stdio.h>
#include <pthread.h>

int num = 0;
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
//pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *reader(void* arg){
        int times = 100000000;
        while(times--){
                pthread_rwlock_rdlock(&rwlock);
                //pthread_mutex_lock(&mutex);
                if(times%1000 == 0)
                        usleep(10);
                pthread_rwlock_unlock(&rwlock);
                //pthread_mutex_unlock(&mutex);
        }
}

void *writer(void* arg){
        int times = 100000000;
        while(times--){
                pthread_rwlock_wrlock(&rwlock);
                //pthread_mutex_lock(&mutex);
                num += 1;
                pthread_rwlock_unlock(&rwlock);
                //pthread_mutex_unlock(&mutex);
        }
}

int main(){
        pthread_t writer_1, reader_1, reader_2;
        pthread_create(&writer_1, NULL, &writer, NULL);
        pthread_create(&reader_1, NULL, &reader, NULL);
        pthread_create(&reader_2, NULL, &reader, NULL);
        pthread_join(writer_1, NULL);
        pthread_join(reader_1, NULL);
        pthread_join(reader_2, NULL);
        printf("num = %d\n", num);
}
```

使用读写锁程序执行时间如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/cost-of-read-write-lock.png)

换成互斥量程序执行时间如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/Mutex%20Overhead.png)

可以明显地看出，对于**多读者少写者**的情况，读写锁要比互斥量效率高一些。

> 其实读写锁也可以解决上边的生产者消费者问题，就是生产者线程和消费者线程都是使用写模式对共享资源加锁。我测试了下，执行时间大概是8秒钟，是互斥量的两倍。所以用读写锁解决生产者消费者问题不仅没必要而且开销大。
>
> 问题的本质在于，生产者消费者问题没有把共享资源的访问作出读和写的细分，无论是生产者还是消费者对共享资源都是写，所以在这个问题里，读写锁发挥不出它的优势，读写锁还是更适合**多读少写**的情况。

# 条件变量

条件变量允许线程睡眠，直到满足某种条件，当满足条件时，可以向该线程发送信号，通知并唤醒该线程。

条件变量通常与互斥量配合一起使用。条件变量由互斥量保护，线程在改变条件状态之前必须首先锁住互斥量，其他线程在获得互斥量之前不会察觉到条件的改变，因为必须在锁住互斥量之后它才可以计算条件是否发生变化。

## POSIX条件变量API

使用条件变量前必须初始化；在释放条件变量的底层内存之前，可以使用pthread_cond_destroy函数进行销毁。

```c
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond,
                      const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
//两函数返回值：成功返回0；出错返回错误编号
```

> 要用默认属性初始化条件变量，只需把attr参数设为NULL。
>
> 对于静态分配的条件变量，也可以使用常量PTHREAD_COND_INITIALIZER来初始化。

```c
#include <pthread.h>
int pthread_cond_wait(pthread_cond_t *restrict cond,
                      pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
                           pthread_mutex_t *restrict mutex,
                           const struct timespec *restrict tsptr);
//两函数返回值：成功返回0；出错返回错误编号
```

- pthread_cond_wait函数用于等待条件变量为真；
- pthread_cond_timedwait函数可以指定一个等待时间，如果在给定时间内条件不能满足，则生成一个返回错误码的变量。

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
//两函数返回值：成功返回0；出错返回错误编号
```

- 以上两个函数用于通知线程条件已满足；
- pthread_cond_signal函数至少能唤醒一个等待该条件的线程；
- pthread_cond_broadcast函数则能唤醒等待该条件的所有线程。

## 条件变量解决生产者消费者问题

生产者消费者问题中，有一个缓冲区大小的概念。

- 如果缓冲区内产品数量为0，则消费者无法消费，消费者线程必须等待；
- 如果缓冲区内产品数量达到最大值，则生产者不应继续生产，生产者线程应该等待。

之前用互斥量解决生产者消费者问题时，并没有考虑这一点。现在有了条件变量就可以解决这个问题了。程序如下：

```c
#include <stdio.h>
#include <pthread.h>

int num = 0;
const int buf_size = 10; //缓冲区大小
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *producer(void* arg){
        while(1){
                pthread_mutex_lock(&mutex);
                while(num >= buf_size){
                        pthread_cond_wait(&cond, &mutex);
                        printf("缓冲区已满，等待消费者消费。\n");
                }
                num += 1;
                printf("生产一个产品，缓冲区当前产品数量：%d\n", num);
                sleep(1); //生产一个产品所需时间
                pthread_cond_signal(&cond);
                printf("通知消费者...\n");
                pthread_mutex_unlock(&mutex);
                sleep(1); //生产产品的频率
        }
}

void *consumer(void* arg){
        while(1){
                pthread_mutex_lock(&mutex);
                while(num <= 0){
                        pthread_cond_wait(&cond, &mutex);
                        printf("缓冲区已空，等待生产者生产。\n");
                }
                num -= 1;
                printf("消费一个产品，缓冲区当前产品数量：%d\n", num);
                sleep(1);
                pthread_cond_signal(&cond);
                printf("通知生产者...\n");
                pthread_mutex_unlock(&mutex);
        }
}

int main(){
        pthread_t producer_thread, consumer_thread;
        pthread_create(&producer_thread, NULL, &producer, NULL);
        pthread_create(&consumer_thread, NULL, &consumer, NULL);
        pthread_join(producer_thread, NULL);
        pthread_join(consumer_thread, NULL);
}
```

程序执行结果：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/cond.png)

# 自旋锁

自旋锁与互斥量类似，但它不使线程进入阻塞态；而是在获取锁之前一直占用CPU，处于忙等（自旋）状态。

自旋锁适用于锁被持有的时间短且线程不希望在重新调度上花费太多成本的情况。

## POSIX自旋锁API

```c
#include <pthread.h>
int pthread_spin_init(pthread_spinlock_t *lock, int pshared);
int pthread_spin_destroy(pthread_spinlock_t *lock);
//两函数返回值：成功返回0；出错返回错误编号
```

- pshared参数表示进程共享属性，表明自旋锁是如何获取的。

```c
#include <pthread.h>
int pthread_spin_lock(pthread_spinlock_t *lock);
int pthread_spin_trylock(pthread_spinlock_t *lock);
int pthread_spin_unlock(pthread_spinlock_t *lock);
//三个函数返回值：成功返回0；出错返回错误编号
```

## 自旋锁解决生产者消费者问题

程序如下：

```c
#include <stdio.h>
#include <pthread.h>

int num = 0;
pthread_spinlock_t spinlock;

void *producer(void* arg){
        int times = 100000000;
        while(times--){
                pthread_spin_lock(&spinlock);
                num += 1;
                pthread_spin_unlock(&spinlock);
        }
}

void *consumer(void* arg){
        int times = 100000000;
        while(times--){
                pthread_spin_lock(&spinlock);
                num -= 1;
                pthread_spin_unlock(&spinlock);
        }
}

int main(){
        pthread_t thread_prod, thread_cons;
        pthread_create(&thread_prod, NULL, &producer, NULL);
        pthread_create(&thread_cons, NULL, &consumer, NULL);
        pthread_join(thread_prod, NULL);
        pthread_join(thread_cons, NULL);
        printf("num = %d\n", num);
}
```

由于我的环境是单核处理器，上边的程序别说1亿次循环，就是10次循环它也没能跑出来。

单核处理器一般建议不要使用自旋锁。因为，在同一时间只有一个线程是处在运行状态，那如果运行线程发现无法获取锁，只能等待解锁，但因为自身不挂起，所以那个获取到锁的线程没有办法进入运行状态，只能等到运行线程把操作系统分给它的时间片用完，才能有机会被调度。这种情况下使用自旋锁的代价很高。

# 屏障

屏障是用户协调多个线程并行工作的同步机制。屏障允许每个线程等待，直到所有的合作线程都到达某一点，然后从该点继续执行。

```c
#include <pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier,
                         const pthread_barrierattr_t *restrict atrr,
                         unsigned int count);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
```

- count参数指定必须到达屏障的线程数目。

```c
#include <pthread.h>
int pthread_barrier_wait(pthread_barrier_t *barrier);
```

- 用pthread_barrier_wait函数来表明线程已完成工作，准备等其他线程赶上来。

> 屏障这部分就不写实例了，一是没想到好的例子；二是屏障用的也比较少。

# 信号量

信号量（Semaphore）本质上是一个计数器，用于为多个进程提供共享数据对象的访问。

为了获得共享资源，进程需要执行下列操作：

- 测试控制该资源的信号量；
- 若信号量值为正，则进程可以使用该资源。在这种情况下，进程会将信号量值减1，表示它使用了一个资源单位；
- 否则，若信号量值为0，则进程进入休眠状态，直至信号量值大于0。进程被唤醒后，返回步骤1。

当进程不再使用该共享资源时，该信号量值增1。如果有进程正在休眠等待此信号量，则唤醒它们。

> 信号量通常是在内核中实现的。

## XSI信号量

使用XSI信号量时，首先通过semget函数获得一个信号量ID。

```c
#include <sys/sem.h>
int semget(key_t key, int nsems, int flags);
//返回值：成功返回信号量ID，出错返回-1
```

- nsems参数指定该集合中的信号量数。

```c
#include <sys/sem.h>
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);
```

- semctl函数包含了多种信号量操作。

```c
#include <sys/sem.h>
int semop(int semid, struct sembuf semoparray[], size_t nops);
//返回值：成功返回0；出错返回-1
```

- semop函数自动执行信号量集合上的操作数组。

## POSIX信号量

POSIX信号量相比XSI信号量有以下优点：

- 性能更高；
- 没有信号量集；
- 删除时更加完美。

POSIX信号量有两种形式：命名的和未命名的。它们的差异在于创建和销毁的形式上。

- 未命名的信号量只存在于内存中，并要求使用信号量的进程必须可以访问内存，这意味着其只能应用于同一进程中的线程，或不同进程中已经映射相同内存内容到它们地址空间中的线程；
- 命名信号量可通过名字访问，因此可以被任何已知它们名字的进程中的线程使用。

```c
#include <semaphore.h>
sem_t *sem_open(const char *name, int oflag, ... /* mode_t mode,
                unsigned int value */);
//成功返回指向信号量的指针；出错返回SEM_FAILED
```

- sem_open函数创建一个新的命名信号量或使用一个现有信号量。

```c
#include <semaphore.h>
int sem_close(sem_t *sem);
//返回值：成功返回0；出错返回-1
```

- sem_close函数用来释放任何信号量相关的资源。

```c
#include <semaphore.h>
int sem_unlink(const char *name);
//成功返回0；出错返回-1
```

- sem_unlink函数销毁命名信号量。如果没有打开的信号量引用，则该信号量会被销毁；否则，销毁将延迟到最后一个打开的引用关闭。

```c
#include <semaphore.h>
int sem_trywait(sem_t *sem);
int sem_wait(sem_t *sem);
//两函数返回值：成功返回0；出错返回-1
```

- sem_wait和sem_trywait函数来实现信号量的减1操作。sem_wait函数会阻塞，sem_trywait函数不会阻塞。

```c
#include <semaphore.h>
#include <time.h>
int sem_timedwait(sem_t *restrict sem,
                  const struct timespec *restrict tsptr);
//成功返回0；出错返回-1
```

- 同样是实现信号量的减1操作，只是可以指定阻塞时间。

```c
#include <semaphore.h>
int sem_post(sem_t *sem);
//成功返回0；出错返回-1
```

- sem_post函数实现信号量的增1操作。

```c
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_destroy(sem_t *sem);
int sem_getvalue(sem_t *restrict sem, int *restrict valp);
//三个函数返回值：成功返回0；出错返回-1
```

- sem_init函数创建一个未命名信号量；pshared参数表明是否在多个进程中使用信号量，如果是则指定一个非0值；value参数指定信号量的初始值。
- sem_destroy函数丢弃信号量；
- sem_getvalue函数用来获取信号量值。

> 如果一个信号量只有值0和1，那它就是二元信号量。当二元信号量是1时，它就是“解锁”的；是0时，它就是“加锁”的。

## POSIX信号量解决生产者消费者问题

对于我们描述的生产者消费者问题，由于生产者和消费者是一个进程内的两个线程，所以我们采用未命名信号量解决该问题。

将信号量初值置为1，即解锁状态。该信号量在执行过程中只有0和1两个值，即二元信号量。程序如下：

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>

int num = 0;
sem_t sem;

void *producer(void* arg){
    int times = 100000000;//循环一亿次
    while(times--){
        sem_wait(&sem);//减1操作，即加锁
        num += 1;//每次生产一个产品
        sem_post(&sem);//增1操作，即解锁
    }
}

void *consumer(void* arg){
    int times = 100000000;
    while(times--){
        sem_wait(&sem);//减1操作，即加锁
        num -= 1;//每次消费一个产品
        sem_post(&sem);//增1操作，即解锁
    }
}

int main(){
    pthread_t thread_prod,thread_cons;
    sem_init(&sem, 0, 1); //信号量初始值置为1,即解锁态
    pthread_create(&thread_prod, NULL, &producer, NULL);
    pthread_create(&thread_cons, NULL, &consumer, NULL);
    pthread_join(thread_prod, NULL);
    pthread_join(thread_cons, NULL);
    sem_destroy(&sem);
    printf("num = %d\n", num);
}
```

执行结果如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/Threads-sync/sem.png)

> 对比可以发现，执行时间比互斥量要长。

# 总结

通过上边对各种同步方式的描述，我们可以做出下述总结。

互斥量（mutex）是最基本的线程同步方式，它只有两种状态（加锁和解锁）。尝试对互斥量加锁的线程如果发现互斥量已经被其他线程上锁了，那该线程就会由运行态进入阻塞态，即让出CPU，CPU可以调度其他线程运行，直到它想要的互斥量被其他线程释放了，CPU就可以把该线程转入就绪态准备调度其运行。

自旋锁与互斥量类似，也是只有解锁和加锁两种状态，它与互斥量的区别在于，它不会阻塞线程。即尝试对自旋锁加锁的线程如果发现自旋锁已经被其他线程上锁了，那该线程将不会让出CPU，会一直处于运行态继续尝试获取该自旋锁，直到它的时间片耗尽让出CPU或者得到锁继续向下执行。

读写锁适用于多读少写的情况，相比于互斥量，它把加锁态细分为读模式加锁和写模式加锁两种，从而允许更高程度的并行，允许多个读者同时访问共享资源，但写者将独占共享资源，所以读写锁也叫共享-互斥锁，即读模式共享，写模式互斥。

信号量相当于对互斥量做了扩展，某种程度上也可以把互斥量看作是特殊的“二元信号量”，当然互斥量更加严格，对于互斥量，解铃还须系铃人，谁锁上的谁负责解开；而二元信号量允许A线程加锁（减1），B线程解锁（增1）。多值信号量允许多个线程同时访问共享资源。

条件变量与互斥量配合使用，主要实现了一种通知机制。
