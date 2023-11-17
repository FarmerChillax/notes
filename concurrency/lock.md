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



### 链接的加载和条件式存储指令

一些平台提供了实现临界区的一对指令：链接的加载（load-lionked）和条件式存储（store-conditional）可以用来配合使用，实现其他并发结构。

```c
int LoadLinked(int *ptr) {
    return *ptr;
}

int StoreConditional(int *ptr, int value) {
    if (no one has updated *ptr since the LoadLinked to this address) {
        *ptr = value;
        return 1; // success!
    } else {
        return 0; // failed to update
    }
}
```

**条件式存储**（store-conditional）指令，只有上一次执行**LoadLinked**的地址在期间**都没有更新**时， 才会成功，同时更新了该地址的值

先通过 **LoadLinked** 尝试获取锁值，如果判断到锁被释放了，就执行**StoreConditional**判断在「执行完」**LoadLinked**到**StoreConditional**「执行前」ptr 有没有被**更新**，没有被更新则说明**没有**其他线程来抢，可以进临界区，有更新则说明已经被其他线程抢走了，继续重复本段落所述内容循环：

```c
void lock(lock_t *lock) {
    while (1) {
        while (LoadLinked(&lock->flag) == 1)
            ; // spin until it's zero
        if (StoreConditional(&lock->flag, 1) == 1)
            return; // if set-it-to-1 was a success: all done
                // otherwise: try it all over again
    }
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```



### 硬件支持：获取并增加

**获取并增加**（fetch-and-add）指令能**原子地返回特定地址的旧值**，并且让该值**自增一**。

```c
int FetchAndAdd(int *ptr) {
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
typedef struct lock_t {
    int ticket;
    int turn;
} lock_t;

void lock_init(lock_t *lock) {
    lock->ticket = 0;
    lock->turn = 0;
}

void lock(lock_t *lock) {
    // 开始挂号，获取当前号码。然后号码加1，保证每个人号码都不同
    int myturn = FetchAndAdd(&lock->ticket);
    // 等待被叫号
    while (lock->turn != myturn)
        ; // spin
}

void unlock(lock_t *lock) {
    // 只需要加1功能，不用返回旧值
    FetchAndAdd(&lock->turn);
}
```

这个方案使用了两个变量来构建锁（ticket和turn），基本操作也很简单：每次进入 lock，就获取当前ticket`值`，相当于**挂号**，然后全局 ticket 本身会**自增一**，因此后续线程都会获得属于自己的唯一 ticket值，**lock->turn**表示当前**叫号值**，叫到号的运行。unlock 时递增lock->turn更新**叫号值**就行（也就是叫下一个号）。这种返回式保证了公平性，相当于每个线程排队运行（FIFO）。

### 自旋过多：怎么办

一个线程会一直自旋检查一个**不会改变**的值，浪费掉整个时间片！如果有 N 个线程去竞争一个锁，情况会更糟糕。同样的场景下，会浪费 **N−1 个时间片**，只是自旋并等待一个线程释放该锁。

如何让锁不会不必要地自旋，浪费 CPU 时间？要解决这个问题，只有硬件支持是不够的，我们还需要操作系统的支持！

### 使用对列：休眠替代自旋

需要一个队列来保存等待锁的线程，上锁时发现锁已被持有，则入队并让调用线程休眠，解锁时从队列中取出一个线程唤醒。Solaris 中 `park()`能够让调用线程休眠，`unpark(threadID)`则会唤醒 threadID 标识的线程。

### 两阶段锁（two-phase lock）

两阶段锁中如果第一个自旋阶段没有获得锁，第二阶段调用者会睡眠，直到锁可用。Linux 中的 futex 就是这种锁，不过只自旋一次；更常见的方式是在循环中自旋固定的次数(希望这段时间内能获取到锁)，然后使用 futex 睡眠。





\
\
\












































