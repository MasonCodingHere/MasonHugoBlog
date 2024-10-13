---
title: "Linux Condition"
description: 
categories:
  - "DefaultCategory"
tags:
  - "Default-Tag"
date: 2024-10-13T02:27:51-07:00
slug: "linux-condition"
math: 
license: 
hidden: false
draft: false
---

## 互斥与同步

在多线程编程中，有两大问题需要解决：**互斥**和**同步**。这两个问题经常放在一起说，但它们还是存在一些差别的。

互斥：由于线程间存在共享数据，当多线程并发地对共享数据进行操作（主要是写操作）时，如不加以管理，可能导致数据不一致问题。互斥就是一个共享数据在同一时刻只能被一个线程使用，这样就保证了共享数据的一致性。

同步：同步比互斥要更加严格。互斥只是规定多个线程不能同时使用共享数据，但是对谁先使用谁后使用并没有作出限制；而同步是指线程间存在依赖，它们应该有严格地执行顺序，如果A还没执行，那B只能等待A执行完再执行。

对于互斥问题，一般用互斥锁（mutex）就可以解决；而同步问题，可以采用条件变量。

> 当然，无论是互斥还是同步，都有其他解决方法，本文只关注互斥锁和条件变量。

## 三线程顺序打印ABC

我们以**三个线程按顺序循环打印字符ABC**为例，在本例中：

- **共享资源**：**标准输出**就是这个问题中的共享资源。

- **互斥问题**：在同一时刻，三个线程中只能有一个打印字符；

- **同步问题**：三个线程之间存在明显的依赖关系：A打印完，B才可以打印；B打印完，C才可以打印；C打印完，A才可以打印。

如果我们仅仅用互斥锁解决互斥问题，即用mutex对标准输出加以保护，确保同一时刻只有一个线程占用标准输出。那如何保证他们按ABC的顺序交替执行打印呢？

你可能说可以通过轮询，比如让B线程一直轮询，一直问A线程：“你打印完没？”，直到获得肯定答案。这样很显然是非常占用CPU时间的，珍贵的CPU时间片全都拿来做轮询了，这是对资源的巨大浪费。

或者你还有另一种方案，让B直接sleep一会儿，等sleep结束，再去问A打印完没。这种方案显然会影响线程的性能。

**条件变量**解决了这个问题。通过条件变量，我们就可以采用**事件模式**。B线程发现A没打印完，就告诉操作系统，我要wait，一会儿会有其他线程发信号来唤醒我的。这个其他线程就是A线程，当A打印完，就调用signal/broadcast,告诉操作系统，之前有线程在wait，现在可以唤醒它（们）了。

> 值得注意的是，条件变量自身并不包含条件，只是它通常与while或if等条件语句搭配使用，故得名条件变量。

**条件变量**、**互斥量**、**用户提供的判定条件**，这三者一般组合使用。
线程检查用户提供的判定条件，如果条件不满足就通过wait函数释放锁然后进入阻塞。这个过程的原子性由条件变量提供，这也是条件变量的意义。

下面贴出**三线程按顺序循环打印字符ABC十次**的源代码：

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int isMyTurn = 0;

void* printA(void* arg){
	for(int i = 0; i < 10; ++i){
		pthread_mutex_lock(&mutex);
		while(isMyTurn != 0){
			pthread_cond_wait(&cond, &mutex);
		}
		printf("A\n");
		sleep(1);
		isMyTurn = 1;
		pthread_cond_broadcast(&cond);
		pthread_mutex_unlock(&mutex);
	}
}

void* printB(void* arg){
	for(int i = 0; i < 10; ++i){
		pthread_mutex_lock(&mutex);
		while(isMyTurn != 1){
			pthread_cond_wait(&cond, &mutex);
		}
		printf("B\n");
		sleep(1);
		isMyTurn = 2;
		pthread_cond_broadcast(&cond);
		pthread_mutex_unlock(&mutex);
	}
}

void* printC(void* arg){
	for(int i = 0; i < 10; ++i){
		pthread_mutex_lock(&mutex);
		while(isMyTurn != 2){
			pthread_cond_wait(&cond, &mutex);
		}
		printf("C\n");
		sleep(1);
		isMyTurn = 0;
		pthread_cond_broadcast(&cond);
		pthread_mutex_unlock(&mutex);
	}
}

int main(){
	pthread_t threadA, threadB, threadC;
	pthread_create(&threadA, NULL, &printA, NULL);
	pthread_create(&threadB, NULL, &printB, NULL);
	pthread_create(&threadC, NULL, &printC, NULL);
	pthread_join(threadA, NULL);
	pthread_join(threadB, NULL);
	pthread_join(threadC, NULL);
}
```

在这个程序里，mutex、cond、isMyTurn这三者就是解决该问题中互斥与同步问题的三要素。

我们通过isMyTurn变量提供判定条件:
- 其值为0时，A可以打印；
- 其值为1时，B可以打印；
- 其值为2时，C可以打印。

以当前B线程获得时间片为例，锁住互斥量之后，检查判定条件，若isMyTurn的值不为1，则调用wait，并将已经锁住的互斥量传递给wait，wait函数会做三件事：
- 对互斥量**解锁**；
- **B线程进入阻塞状态**，即把B线程放到等待cond条件变量的列表里。
- wait函数返回时，对互斥量**重新加锁**。

由于B线程进入阻塞状态，调度程序会选择其他就绪线程执行，我们假设是A线程被调度，则A首先锁住互斥量，然后检查判定条件，发现isMyTurn当前值为0，所以它打印字符'A',并将isMyTurn置为1，然后调用signal/broadcast函数通知操作系统，可以唤醒正在等待cond条件变量的线程了，使它（们）脱离阻塞状态。

被唤醒的线程从wait函数返回，返回的同时会自动给互斥量加锁，然后继续检查判定条件，若不是自己的值，则继续调用wait进入阻塞；若是自己对应的值，则打印对应字符，并改变isMyTurn的值。

## 存在的问题

### signal和broadcast用哪个好？

signal和broadcast函数区别如下：

- signal函数只唤醒一个等待该条件的线程；当有多个线程等待cond时，则唤醒优先级最高的一个；若多个等待cond的线程优先级相同，则唤醒谁是不确定的。
- broadcast函数则唤醒所有等待该条件的线程。

适合用broadcast的情况：

- 读者写者问题。即写者写完后，通知所有读线程可以读共享数据了；
- 一个生产者多个消费者且生产者一次生产多个产品的情况。生产者生产完，通知所有消费者可以消费了；
- 多生产者多消费者问题。

适合用signal的情况：

- 单一生产者且每次生产一个产品的情况，最好只有一个消费者。

### signal/broadcast与unlock的顺序问题

首先强调，man手册中明确指明：

> The pthread_cond_broadcast() or pthread_cond_signal() functions may be called by a thread whether or not it currently owns the mutex  that  threads calling  pthread_cond_wait()  or  pthread_cond_timedwait()  have associated with the condition variable during their waits; however, if predictable scheduling behavior is required, then that mutex shall be locked by the thread calling pthread_cond_broadcast() or pthread_cond_signal().

即无论当前线程是否持有锁，它都可以调用signal/broadcast。尽管如此，如果需要可预测的调度行为，还是应该在上锁的情况下调用signal和broadcast。

> 但是在网上看到，上锁情况下调用signal和broadcast会降低效率。
>
> signal/broadcast之后，unlock之前，可能其他线程已经获得时间片，它想锁住互斥量，但发现互斥量还被锁着，只能继续阻塞，直到持有锁的线程执行unlock后，它再次获得时间片，才能上锁。

针对上面的**三线程顺序打印ABC**的程序，我进行了测试，测试环境为**单核机器**，测试结果如下：

|      函数选用       |        测试结果         |
| :-----------------: | :---------------------: |
|  先signal再unlock   | 打印完一次ABC后卡住不动 |
|  先unlock再signal   |          正常           |
| 先broadcast再unlock |          正常           |
| 先unlock再broadcast |          正常           |

> 为什么先signal再unlock，执行结果不对，这个问题有待解决，还没有想通。欢迎知道的朋友留言，教教我，谢谢。







# References
> 1. 
