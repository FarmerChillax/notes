# 第二十八章-锁



并发编程要解决的最基本的问题：**我们希望以原子方式执行一系列的指令**，但由于中断的存在我们做不到这点。因此本章介绍了锁（lock）来解决这一问题。程序员在代码中加锁，放在临界区周围，保证临界区能够像单条原子指令一样执行。



### 锁的基本思想 <a href="#suo-de-ji-ben-si-xiang" id="suo-de-ji-ben-si-xiang"></a>

lock()和 unlock()函数的语义很简单。调用 **lock()** 尝试获取锁，如果没有其他线程持有锁（即它是**可用**`的`），该线程会**获得锁**，进入**临界区**。当持有锁的线程在临界区时，其他线程就无法进入临界区。伪代码如下：

```bash
lock_t mutex;
...
lock(&mutex);
// 临界区代码(可能存在并发问题的代码）
// 比如全局变量的操作: balance = balance + 1
... 
unlock(&mutex);
```

#### Pthread 锁

POSIX 库将锁称为互斥量（mutex）使用代码如下：

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&lock); // wrapper for pthread_mutex_lock()
balance = balance + 1;
pthread_mutex_unlock(&lock);
```



#### 实现一个锁

我们从程序员的角度，对锁如何工作有了一定的了解，那么如何实现一个锁呢？需要硬件提供什么支持？操作系统需要提供什么支持？本章后面将回答这些问题，现在我们来想想「如何实现一个**高效**锁」

* 如何实现一个「高效」的锁？
* 需要



### 评价锁的几个指标

































