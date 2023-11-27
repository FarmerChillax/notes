# 第三十二章-常见并发问题

从下面的统计中我们可以看出，常见的并发问题可以分为：「非死锁的缺陷」与「死锁缺陷」

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

### 非死锁缺陷

非死锁缺陷主要可以概括为以下几个：

* 违反原子性

「违反原子性」是指违反了多次内存访问中预期的**可串行性**，也就是说临界区代码没有被锁保护起来（即代码段本意是原子的，但在执行中并没有强制实现原子性）。如下面的代码，解决方法也就是加锁:

```c
Thread 1::
    // 判断和赋值应该是原子性的
    if (thd->proc_info)
    {
        ...
        fputs(thd->proc_info, ...);
        ...
    }

Thread 2::
    thd->proc_info = NULL;
```

* 违反顺序性

两个内存访问的**预期顺序**被打破了（即 A 应该在 B 之前执行，但是实际运行中却不是这个顺序）比如下面的这段代码中，如果 mState = mThread->State 语句先执行，则 mThread 为空。可以通过加**条件变量**(或信号量)解决：

```c
Thread 1::
    void init()
    {
        ...
        mThread = PR_CreateThread(mMain, ...);
        ...
    }

Thread 2::
    void mMain(...)
    {
        ...
        mState = mThread->State;
        ...
    }
```



### 死锁缺陷

除了上面的缺陷外，死锁（deadlock）死一种并发系统中经典的问题。比如下面的代码就可能会出现死锁：

```c
Thread 1:
lock(L1);
lock(L2);

Thread 2:
lock(L2);
lock(L1);
```

#### 为什么发生死锁

* 复杂的依赖：以操作系统为例。虚拟内存系统依赖文件系统才能从磁盘读到内存页；`文件系统`依赖`虚拟内存系统`申请一页内存，以便存放读到的块。
* 封装：软件开发者一直倾向于隐藏实现细节，以模块化的方式让软件开发更容易。然而，模块化和锁不是很契合，以 Java 的 Vector 类和 AddAll()方法为例，我们这样调用这个方法：

```java
Vector v1, v2;
v1.AddAll(v2);
```

在内部，这个方法需要多线程安全，因此针对被添加向量（v1）和参数（v2）的锁都需要获取。假设这个方法，先给 v1 加锁，然后再给 v2 加锁。如果另外某个线程几乎同时在调用 v2.AddAll(v1)，就可能遇到死锁。

#### 产生死锁的条件

* 互斥：线程对于需要的资源进行互斥的访问（例如一个线程抢到锁）
* 持有并等待：线程持有了资源（例如已将持有的锁），同时又在等待其他资源（例如，需要获得的锁）
* 非抢占：线程获得的资源（例如锁），不能被抢占
* 循环等待：线程之间存在一个环路，环路上每个线程都额外持有一个资源，而这个资源又是下一个线程要申请的

#### 预防死锁

* 循环等待：这个方法是最实用和常用的方法，就是让代码不产生循环等待的情况。最直接的方法就是获取锁的时候提供一个**全序**（total ordering），通过严格的加锁顺序避免了循环等待。
* 持有并等待：这个方法通过原子地抢锁来避免死锁问题，也就是说在用一把大锁把几个锁保护起来，但这么做可能会影响性能。
* 非抢占：解决非抢占就是要让线程能占用其他线程没在用的锁，不能占着茅坑不拉屎。具体来说`trylock()` 函数会尝试获取锁，当锁被占有时则返回-1。
* 互斥：如果不存在锁那么便不存在死锁。通过强大的硬件指令，我们可以构造出**不需要锁**的数据结构，这个强大的硬件指令就是「比较并交换（compare-and-swap）」。

下面是一个「比较并交换」的例子，在 `*address` 的值等于 `expected` 值时，将其赋值为 new：

```c
int CompareAndSwap(int *address, int expected, int new)
{
    if (*address == expected)
    {
        *address = new;
        return 1; // success
    }
    return 0; // failure
}
```

假定我们想原子地给某个值增加特定的数量:

```c
void AtomicIncrement(int *value, int amount)
{
    do
    {
        int old = *value;
    } while (CompareAndSwap(value, old, old + amount) == 0);

    /** 无同步操作
        int old = *value;
        *value = old + amount;
    */
}
```

一个更复杂的例子：**链表插入**。这是在链表头部插入元素的代码：

```c
void insert(int value)
{
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    n->next = head;
    head = n;
}
```

通过**加锁**解决同步：

```c
void insert(int value)
{
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    lock(listlock); // begin critical section
    n->next = head;
    head = n;
    unlock(listlock); // end of critical section
}
```

用**比较并交换指令**（compare-and-swap)来实现插入操作:

```c
void insert(int value)
{
    node_t *n = malloc(sizeof(node_t));
    assert(n != NULL);
    n->value = value;
    do
    {
        n->next = head;
    } while (CompareAndSwap(&head, n->next, n) == 0);
}
```

这段代码，首先把 next 指针指向当前的链表头（head），然后试着把新结点交换到链表头。但是，如果此时其他的线程成功地修改了 head 的值，这里的交换就会失败，导致这个线程根据新的 head 值重试。



#### 通过调度避免死锁

除了死锁预防外，有的场景死锁避免（avoidance）可能会更加合适，通过合理的调度避免产生死锁。比如下面的这个例子中，只要 T1 和 T2 (需要同一个锁的线程)不同时运行，就不会产生死锁。但这样就失去了并发性

[![F32.1](https://hjk.life/assets/img/2022-06-16-operating-systems-26/F32.1.jpg)](https://hjk.life/assets/img/2022-06-16-operating-systems-26/F32.1.jpg)

\


#### 检查和恢复

最后一种常用的策略就是允许死锁偶尔发生，检查到死锁时再采取行动。

> 提示：不要总是完美（TOM WEST 定律）
>
> Tom West 是经典的计算机行业小说《Soul of a New Machine》的主人公，有一句很棒的工程格言：“不是所有值得做的事情都值得做好”。`如果坏事很少发生，并且造成的影响很小，那么我们不应该去花费大量的精力去预防它`。当然，如果你在制造航天飞机，事故会导致航天飞机爆炸，那么你应该忽略这个建议。

很多数据库系统使用了**死锁检测和恢复技术**。死锁检测器会定期运行，通过构建资源图来检查循环。当循环（死锁）发生时，系统需要重启。如果还需要更复杂的数据结构相关的修复，那么需要人工参与。















