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



### 评价锁的几个指标

* 能否完成它的基本任务——互斥（mutual exclusion）
* 公平性（fairness）——当锁可用时，是否每个竞争的线程都有公平的机会抢到锁？
* 性能（performance）——锁竞争带来的时间开销，比如单 CPU 上竞争的开销的和多 CPU 上多个线程竞争时的开销，通过比较不同的场景，我们嗯更好的理解不同锁技术对性能的影响。

### 控制中断

控制中断是最早提供的互斥解决方案之一，其本质就是在执行临界区代码的时候关闭中断。这个方案是为单处理器系统开发的，代码如下：

```c
void lock() {
    DisableInterrupts();
}
void unlock() {
    EnableInterrupts();
}
```

这样的方案的优点是简单、清晰易懂。但缺点却很多：

1. 要**允许所有调用线程执行特权操作**（打开/关闭中断），信任这种机制不会被滥用。但这是理想的情况，因为我们无法得知用户允许的程序是否可信任，因此该方案不适合作为通用方案。糟糕的情况比如有以下几个：
   1. 一个贪婪的程序可以在开始时就关闭中断，从而独占处理器。
   2. 更糟糕的情况是被恶意程序利用，调用 lock 后一直死循环，系统无法重新获得控制权，只能重启系统。
2. **不支持多处理器**，如果多个线程允许在不同的 CPU 上，每个线程都试图进入同一个临界区，那么即便关中断也没用。当下多处理器已经很普遍了，因此需要方案更加通用。
3. **关中断导致中断丢失**，这可能会导致严重的系统问题。比如磁盘完成了读取请求，正常应该会给个中断，但 CPU 因为关中断导致错失了这一中断，那么，操作系统如何知道去唤醒等待读取的进程？
4. **效率低**，与正常的执行指令相比，现代 CPU 对于关闭/打开中断的代码执行的比较慢。

因此基于上述的情况，只在少数的情况下用开/关中断来实现互斥原语，比如操作系统本身会采用屏蔽中断的方式，保证访问自己数据结构的原子性或者避免复杂的中断处理情况，因为在操作系统内部它总是可信的，不存在信任问题。

> 「原语」是什么？个人简单的理解，原语是指**操作系统层面**提供的指令，一个原语中可能有很多个底层的指令，但操作系统帮我们确保了这些指令以原子的方式执行。

### 「硬件支持」测试并设置指令（原子交换）

锁的实现需要硬件支持，最简单的**硬件支持**是**测试并设置指令**（test-and-set instruction，TAS），也叫作**原子交换**（atomic exchange）。

```c
typedef struct lock_t { int flag; } lock_t;

void init(lock_t *mutex) {
    // 0 -> lock is available, 1 -> held
    mutex->flag = 0;
}

void lock(lock_t *mutex) {
    while (mutex->flag == 1) // TEST the flag
        ; // spin-wait (do nothing)
    mutex->flag = 1; // now SET it!
}

void unlock(lock_t *mutex) {
    mutex->flag = 0;
}
```

上述代码中，每次 lock 会判断 flag 的值，也就是**测试并设置指令（test-and-set instruction）**中的test，然后判断成功才 **set**

但如果没有硬件辅助，也就是让**测试并设置**作为一个原子操作，会导致两个线程有可能**同时进入**临界区。

> 注意自旋等待spin-wait会影响性能

### 实现可用的自旋锁

自旋锁（spin lock）其实就是一直自旋（判断某个条件成立），直到锁可用。

在 SPARC 上，需要的指令叫 ldstub（load/store unsigned byte，加载/保存无符号字节）；在 x86 上，是 xchg（atomic exchange，原子交换）指令。但它们基本上在不同的平台上做同样的事，通常称为**测试并设置指令**（test-and-set）。

```c
int TestAndSet(int *old_ptr, int new) {
    int old = *old_ptr; // fetch old value at old_ptr
    *old_ptr = new; // store 'new' into old_ptr
    return old; // return the old value
}
```

将**测试并设置**作为**原子操作**：

```c
typedef struct lock_t {
    int flag;
} lock_t;

void init(lock_t *lock) {
    // 0 indicates that lock is available, 1 that it is ld
    lock->flag = 0;
}

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        ; // spin-wait (do nothing)
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```

#### 评价自旋锁

现在我们来用之前提到的标准来评价基本的自旋锁：

* 锁最重要的一点是正确性（correctness），也就是能够互斥吗？答案是肯定的，因此自旋锁是一把正确的锁。
* 公平性（fairness）：显然自旋锁不提供任何的公平性保证，因此可能会导致饿死。
* 性能（performan）：单处理器上，性能开销相当大。因为在上锁后最少会自旋一个时间片，浪费 CPU 周期。在多处理器上，自旋锁性能不错。因为临界区一般很短，因此锁很快就可用了（自旋的时候），并没有浪费很多 CPU 周期，因此效果还不错。



### 硬件支持：比较并交换指令（CAS） <a href="#ying-jian-zhi-chi-bi-jiao-bing-jiao-huan-zhi-ling" id="ying-jian-zhi-chi-bi-jiao-bing-jiao-huan-zhi-ling"></a>

某些系统提供了另一个硬件原语——比较交换指令（SPARC 中是 compare-and-swap，x86 中是 compare-and-exchange）。下面是伪代码：

```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;
}
```

可以看到和**测试并设置指令**的工作方式类似，但其实它十分强大，这在后面讨论「无等待同步（wait-free synchronization）时，会用到这条指令的强大之处。但如果只是用它实现一个简单的自旋锁，那么它无疑等价于上面分析的自旋锁。



























