---
title: Linux多线程
date: 2016-03-10 19:37:07
tags: 多线程
categories: Linux
---

## Linux环境多线程线程开发API
多线程开发在Linux平台上的支持库是Pthread。其中设计的多线程开发的最基本的概念主要包含：线程操作，互斥锁，条件。线程操作主要包线程的创建，退出，等待三种。互斥锁包括四种操作，分别是创建，销毁，加锁和解锁。条件操作有5种操作：创建，销毁，触发，广播和等待。

**Linux Pthread API**

```  
线程操作：  
创建	pthread_create	
退出	pthread_exit  
等待	pthread_join	

互斥锁:	
创建	pthread_mutex_init	
销毁	pthread_mutex_destroy	
加锁	pthread_mutex_lock	
解锁	pthread_mutex_unlock	

条件变量:	
创建	pthread_cond_init	
销毁	pthread_cond_destroy	
触发	pthread_cond_signal	  
广播	pthread_cond_broadcast	S  
等待	pthread_cond_wait / pthread_cond_timedwait
```
<!--more-->
### 线程操作
**线程操作example：**


```

\# include <stdio.h>
\# include <stdlib.h>
\# include <pthread.h>

void *
thr_fn1(void *arg)
{
    printf("thread 1 returning\n");
    return((void *)1);
}

void *
thr_fn2(void *arg)
{
    printf("thread 2 exiting\n");
    pthread_exit((void *)2);
}

int
main(void)
{
    int         err;
    pthread_t   tid1, tid2;
    void        *tret;

    err = pthread_create(&tid1, NULL, thr_fn1, NULL);
    if (err != 0)
        err_exit(err, "can't create thread 1");
    err = pthread_create(&tid2, NULL, thr_fn2, NULL);
    if (err != 0)
        err_exit(err, "can't create thread 2");
    err = pthread_join(tid1, &tret);
    if (err != 0)
        err_exit(err, "can't join with thread 1");
    printf("thread 1 exit code %ld\n", (long)tret);
    err = pthread_join(tid2, &tret);
    if (err != 0)
        err_exit(err, "can't join with thread 2");
    printf("thread 2 exit code %ld\n", (long)tret);
    exit(0);
} 

执行结果：
thread 1 turning
thread 2 exiting
thread 2 exit code 1
thread 2 exit code 2

```

进程中的其它进程可以通过调用pthread_joi函数获得该线程的退出状态。当pthread_exit函数传递包含复杂信息的结构体的地址时，要注意这个结构所使用的内存在调用者完成调用以后仍是有效的。不能在线程的栈空间上分配该结构体，可以使用全局结构体或者用malloc函数分配空间。
### 线程同步
当多个控制线程共享相同的内存时，需要确保每个线程看到一致的数据视图。当一个线程可以修改的变量，其它线程也可以读取或者修改的时候，就需要对这些线程进行同步，确保它们在访问变量的存储内容时不会访问到无效的值。

```
\#include <stdlib.h>
\#include <pthread.h>

struct foo {
    int             f_count;
    pthread_mutex_t f_lock;
    int             f_id;
    /* ... more stuff here ... */
};

struct foo *
foo_alloc(int id) /* allocate the object */
{
    struct foo *fp;

    if ((fp = malloc(sizeof(struct foo))) != NULL) {
        fp->f_count = 1;
        fp->f_id = id;
        if (pthread_mutex_init(&fp->f_lock, NULL) != 0) {
            free(fp);
            return(NULL);
        }
        /* ... continue initialization ... */
    }
    return(fp);
}

void
foo_hold(struct foo *fp) /* add a reference to the object */
{
    pthread_mutex_lock(&fp->f_lock);
    fp->f_count++;
    pthread_mutex_unlock(&fp->f_lock);
}

void
foo_rele(struct foo *fp) /* release a reference to the object */
{
    pthread_mutex_lock(&fp->f_lock);
    if (--fp->f_count == 0) { /* last reference */
        pthread_mutex_unlock(&fp->f_lock);
        pthread_mutex_destroy(&fp->f_lock);
        free(fp);
    } else {
        pthread_mutex_unlock(&fp->f_lock);
    }
 ｝
 
```
例子中用于保护某个数据的互斥量。当一个以上的线程需要访问动态分配的对象时，在对象中嵌入引入计数，确保在所有使用该对象的线程完成数据访问之前，该对象不会被释放。

在对饮用计数加1，减1，检查引用计数是否达到0，这些操作之前需要锁住互斥量。使用该对象前，线程需要调用foo_hold对这个对象的引用计数加1.当对象使用完毕时，必须调用foo_rele释放引用。最后一个引用释放时，对象所占的内存空间就被释放。

当存在多个互斥量时，为了避免死锁产生，可以通过控制互斥量的加锁顺序来避免死锁的发生。例如，假设需要对两个互斥量A和B同时加锁。如果所有线程总是对互斥量B加锁之前锁住互斥量A，那么使用这两个互斥量就不会产生死锁。

## 条件变量
条件变量是线程可用的另一种同步机制。条件变量给多个线程提供了一个回合的场所。条件变量与互斥量一起使用，允许线程以无竞争的方式等待特定的条件发生。

条件本身是由互斥量保护的。线程在改变条件状态之前必须首先锁住互斥量。传递给pthread_cond_wait的互斥量对条件进行了保护。调用者把锁住的互斥量传给函数，函数然后自动把调用线程放到等待条件列表上，对互斥量解锁。pthread_cond_wait返回时，互斥量会再次被锁住。

```

 #include <pthread.h>

 struct msg {
     struct msg *m_next;
     /* ... more stuff here ... */
 };

 struct msg *workq;

 pthread_cond_t qready = PTHREAD_COND_INITIALIZER;

 pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;

 void
 process_msg(void)
 {
     struct msg *mp;

     for (;;) {
         pthread_mutex_lock(&qlock);
         while (workq == NULL)
             pthread_cond_wait(&qready, &qlock);
         mp = workq;
         workq = mp->m_next;
         pthread_mutex_unlock(&qlock);
         /* now process the message mp */
     }
 }

 void
 enqueue_msg(struct msg *mp)
 {
     pthread_mutex_lock(&qlock);
     mp->m_next = workq;
     workq = mp;
     pthread_mutex_unlock(&qlock);
     pthread_cond_signal(&qready);
 }
 
```
条件是工作队列的状态。用互斥量保护条件，在while循环中判断条件。把消息放倒工作队列时需要占有互斥量，但在等待线程发信号时，不需要占有互斥量。只要线程在调用pthread_cond_signal之前把消息从队列中拖出了，就可以释放互斥量已完成这部分工作。