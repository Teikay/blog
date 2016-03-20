---
title: 生产者消费者模型
date: 2016-03-20 12:04:13
tags: 多线程
categories: Linux
---

## 问题描述
生产者消费者问题（英语：Producer-consumer problem），也称有限缓冲问题（英语：Bounded-buffer problem），是一个多线程同步问题的经典案例。该问题描述了两个共享固定大小缓冲区的线程——即所谓的“生产者”和“消费者”——在实际运行时会发生的问题。生产者的主要作用是生成一定量的数据放到缓冲区中，然后重复此过程。与此同时，消费者也在缓冲区消耗这些数据。该问题的关键就是要保证生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时消耗数据。  

要解决该问题，就必须让生产者在缓冲区满时休眠（要么干脆就放弃数据），等到下次消费者消耗缓冲区中的数据的时候，生产者才能被唤醒，开始往缓冲区添加数据。同样，也可以让消费者在缓冲区空时进入休眠，等到生产者往缓冲区添加数据之后，再唤醒消费者。

<!--more-->

## 问题解决方法

** 线程条件变量：** pthread_cond_t  
 
** 线程等待某个条件：**  
int pthread_cond_timedwait(pthread_cond_t *restrict cond,pthread_mutex_t *restrict mutex,const struct timespec *restrict abstime);  
int pthread_cond_wait(pthread_cond_t *restrict cond,pthread_mutex_t *restrict mutex);    
 
* 首先释放锁  
* 等待，直到接收到signal  
* 重新抢占锁  

** 通知函数 **  
 
* 通知所有的线程：int pthread_cond_broadcast(pthread_cond_t *cond);   
* 只通知一个线程：int pthread_cond_signal(pthread_cond_t *cond); 

代码示例：

```  
#include "queue.h"
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#define BUF_SIZE 3

pthread_mutex_t mutex;    //  互斥锁 保护队列
pthread_cond_t full;    //同步量  有产品可取
pthread_cond_t empty;   //同步量 有空位可放
Queue Q;

void *producer(void* arg){
    while(1){
        pthread_mutex_lock(&mutex);
        while(Q.size_ == BUF_SIZE){
            pthread_cond_wait(&empty, &mutex);
        }
        int data = rand()%100;
        printf("produce a %d\n", data);
        queue_push(&Q, data);
        pthread_cond_signal(&full);
        pthread_mutex_unlock(&mutex);
    }
}

void *consumer(void* arg){
    while(1){
        pthread_mutex_lock(&mutex);
        while(queue_is_empty(&Q)){ 
        //如果采用if判断条件的会产生消费者去空队列的情况，如果一个等待线程被激活，然后尚未拿到锁，
        //另一个线程抢到锁，直接绕过if语句（此时队列不为空），然后将数据消费，此时队列为空。
        //之前的线程拿到锁，继续消费导致错误。
            printf("wait for producer\n");
            pthread_cond_wait(&full, &mutex);
        }
        int data = queue_top(&Q);
        printf("consume a %d\n", data);
        queue_pop(&Q);
        pthread_cond_signal(&empty);
        pthread_mutex_unlock(&mutex);
    }
}

int main(int argc, const char *argv[])
{
    srand(100000);
    pthread_t tid1, tid2, tid3;
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&full, NULL);
    pthread_cond_init(&empty, NULL);
    queue_init(&Q);

    pthread_create(&tid3, NULL, producer, NULL);
    pthread_create(&tid1, NULL, consumer, NULL);
    pthread_create(&tid2, NULL, consumer, NULL);

    queue_destroy(&Q);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&full);
    pthread_cond_destroy(&empty);
    return 0;
}

```  
pthread_cond_wait和pthread_cond_signal
关于其内部实现伪代码如下：

``` 
pthread_cond_wait(mutex, cond):
    value = cond->value; /* 1 */
    pthread_mutex_unlock(mutex); /* 2 */
    pthread_mutex_lock(cond->mutex); /* 10 */    pthread_cond_t自带一个mutex来互斥对waiter等待链表的操作
    if (value == cond->value) { /* 11 */    检查一次是不是cond有被其他线程设置过，相当于单例模式的第二次检测是否为NULL
        me->next_cond = cond->waiter;
        cond->waiter = me;//链表操作
        pthread_mutex_unlock(cond->mutex);
        unable_to_run(me);
    } else
        pthread_mutex_unlock(cond->mutex); /* 12 */
    pthread_mutex_lock(mutex); /* 13 */    
    
pthread_cond_signal(cond):
    pthread_mutex_lock(cond->mutex); /* 3 */
    cond->value++; /* 4 */
    if (cond->waiter) { /* 5 */
        sleeper = cond->waiter; /* 6 */
        cond->waiter = sleeper->next_cond; /* 7 */        //链表操作
        able_to_run(sleeper); /* 8 */    运行sleep的线程，即上面的me
    }
    pthread_mutex_unlock(cond->mutex); /* 9 */
    
 ``` 
 
 以上