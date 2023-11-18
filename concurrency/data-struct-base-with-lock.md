# 第二十九章-基于锁的并发数据结构

通过锁可以使数据结构线程安全（thread safe），但具体如何加锁则决定了该数据结构的效率。本章将探讨怎么给数据结构加锁，才能让该结构功能正确的同时保证高性能。

### 并发计数器

下面先看一个简单的**非并发的**计数器，然后一步步的在此基础上构建一个并发安全且高性能的计数器：

```c
typedef struct counter_t
{
    int value;
} counter_t;

void init(counter_t *c)
{
    c->value = 0;
}

void increment(counter_t *c)
{
    c->value++;
}

void decrement(counter_t *c)
{
    c->value--;
}

int get(counter_t *c)
{
    return c->value;
}
```

可以看到计数器的代码非常简单，而我们想其并发安全显而易见的方法便是在结构体中添加一把锁，在数据做修改操作的临界区代码添加上这把锁，修改后的代码如下：

```c
typedef struct counter_t
{
    int value;
    pthread_mutex_t lock;
} counter_t;

void init(counter_t *c)
{
    c->value = 0;
    Pthread_mutex_init(&c->lock, NULL);
}

void increment(counter_t *c)
{
    Pthread_mutex_lock(&c->lock);
    c->value++;
    Pthread_mutex_unlock(&c->lock);
}

void decrement(counter_t *c)
{
    Pthread_mutex_lock(&c->lock);
    c->value--;
    Pthread_mutex_unlock(&c->lock);
}

int get(counter_t *c)
{
    Pthread_mutex_lock(&c->lock);
    int rc = c->value;
    Pthread_mutex_unlock(&c->lock);
    return rc;
}
```

这样做在多CPU环境下性能很差，因为这种锁导致了多 CPU 情况下也只允许一个线程在运行，其他都在自旋等待，没发挥出多 CPU 的优势。相当于退化成串行，并且还增加了锁的开销。**理想情况下，虽然工作量增多，但并行执行后，完成任务的时间并没有增加。**

#### 可扩展的计数器（扩展的意思是支持多 CPU） <a href="#ke-kuo-zhan-de-ji-shu-kuo-zhan-de-yi-si-shi-zhi-chi-duo-cpu" id="ke-kuo-zhan-de-ji-shu-kuo-zhan-de-yi-si-shi-zhi-chi-duo-cpu"></a>

为了解决上面的问题，有一种解决方法称为：懒惰计数器（sloppy counter）

懒惰计数器通过多个**局部计数器**和一个**全局计数器**来实现一个逻辑计数器，其中每个 CPU 核心有一个局部计数器。具体来说，在 4 个 CPU 的机器上，有 4 个局部计数器和 1 个全局计数器。除了这些计数器，还有锁：每个局部计数器有一个锁，全局计数器有一个。

懒惰计数器的基本思想是这样的。如果一个核心上的线程想增加计数器，那就增加它的局部计数器，访问这个局部计数器是通过对应的**局部锁**同步的。因为每个 CPU 有自己的局部计数器，不同 CPU 上的线程不会竞争，所以计数器的更新操作可扩展性好。

为了保持全局计数器更新（以防某个线程要读取该值），**局部值会定期转移给全局计数器**，方法是获取**全局锁**，让全局计数器加上局部计数器的值，然后将局部计数器置零

**局部转全局的频度，取决于一个阈值**，这里称为 S（表示 sloppiness）。S 越小，懒惰计数器则越趋近于非扩展的计数器。S 越大，扩展性越强，但是全局计数器与实际计数的偏差越大。

在这个例子中，阈值 S 设置为 5，4 个 CPU 上分别有一个线程更新局部计数器 L1,…, L4。随着时间增加，全局计数器 G 的值也会记录下来。每一段时间，局部计数器可能会增加。如果局部计数值增加到阈值 S，就把局部值转移到全局计数器，局部计数器清零。

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

```c
typedef struct counter_t
{
    int global;                     // global count
    pthread_mutex_t glock;          // global lock
    int local[NUMCPUS];             // local count (per cpu)
    pthread_mutex_t llock[NUMCPUS]; // ... and locks
    int threshold;                  // update frequency
} counter_t;

// init: record threshold, init locks, init values
// of all local counts and global count
void init(counter_t *c, int threshold)
{
    c->threshold = threshold;

    c->global = 0;
    pthread_mutex_init(&c->glock, NULL);

    int i;
    for (i = 0; i < NUMCPUS; i++)
    {
        c->local[i] = 0;
        pthread_mutex_init(&c->llock[i], NULL);
    }
}

// update: usually, just grab local lock and update local amount
// once local count has risen by 'threshold', grab global
// lock and transfer local values to it
void update(counter_t *c, int threadID, int amt)
{
    pthread_mutex_lock(&c->llock[threadID]);
    c->local[threadID] += amt; // assumes amt > 0
    if (c->local[threadID] >= c->threshold)
    { // transfer to global
        pthread_mutex_lock(&c->glock);
        c->global += c->local[threadID];
        pthread_mutex_unlock(&c->glock);
        c->local[threadID] = 0;
    }
    pthread_mutex_unlock(&c->llock[threadID]);
}

// get: just return global amount (which may not be perfect)
int get(counter_t *c)
{
    pthread_mutex_lock(&c->glock);
    int val = c->global;
    pthread_mutex_unlock(&c->glock);
    return val; // only approximate!
}
```

### 并发链表

我们和上面一样，先添加一把大锁保证了并发安全，然后再进一步探讨如何优化性能：

```c
// basic node structure
typedef struct node_t
{
    int key;
    struct node_t *next;
} node_t;

// basic list structure (one used per list)
typedef struct list_t
{
    node_t *head;
    pthread_mutex_t lock;
} list_t;

void List_Init(list_t *L)
{
    L->head = NULL;
    pthread_mutex_init(&L->lock, NULL);
}

int List_Insert(list_t *L, int key)
{
    pthread_mutex_lock(&L->lock);
    node_t *new = malloc(sizeof(node_t));
    if (new == NULL)
    {
        perror("malloc");
        pthread_mutex_unlock(&L->lock);
        return -1; // fail
    }
    new->key = key;
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
    return 0; // success
}

int List_Lookup(list_t *L, int key)
{
    pthread_mutex_lock(&L->lock);
    node_t *curr = L->head;
    while (curr)
    {
        if (curr->key == key)
        {
            pthread_mutex_unlock(&L->lock);
            return 0; // success
        }
        curr = curr->next;
    }
    pthread_mutex_unlock(&L->lock);
    return -1; // failure
}
```

从代码中可以看出，**插入函数**入口处获取锁，结束时释放锁。如果 malloc 失败（在极少的时候），会有一点小问题，在这种情况下，代码在插入失败之前，必须释放锁。

我们调整代码，让获取锁和释放锁只环绕插入代码的真正临界区(缩小临界区)。前面的方法有效是因为部分工作实际上不需要锁，假定 malloc()是线程安全的，每个线程都可以调用它，不需要担心竞争条件和其他并发缺陷。只有在更新共享列表时需要持有锁。下面展示了这些修改的细节。

```c
void List_Init(list_t *L)
{
    L->head = NULL;
    pthread_mutex_init(&L->lock, NULL);
}

void List_Insert(list_t *L, int key)
{
    // synchronization not needed
    node_t *new = malloc(sizeof(node_t));
    if (new == NULL)
    {
        perror("malloc");
        return;
    }
    new->key = key;

    // just lock critical section
    pthread_mutex_lock(&L->lock);
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
}

int List_Lookup(list_t *L, int key)
{
    int rv = -1;
    pthread_mutex_lock(&L->lock);
    node_t *curr = L->head;
    while (curr)
    {
        if (curr->key == key)
        {
            rv = 0;
            break;
        }
        curr = curr->next;
    }
    pthread_mutex_unlock(&L->lock);
    return rv; // now both success and failure
}
```

现在已经能保证这个链表的并发安全了，想要解决性能问题，可以使用一种叫过手锁（hand-over-hand locking，也叫锁耦合，lock coupling），其原理十分简单，就是每个节点都有一个锁，替代之前整个链表一个锁。遍历链表的时候，首先抢占下一个节点的锁，然后释放当前节点的锁。

从理论上来说确实有点合理，但实际上在遍历的时候，每个节点都要加锁、解锁，而这个开销也是十分巨大的，很难说比单锁的方法快。即使有大量的线程和很大的链表，也不一定就比单锁快，它只适用于十分单一且特殊的场景，或者夹杂在某些方案里面。

### 并发队列

现在我们知道可以用标准的方法来创建一个并发安全的数据结构：添加一把大锁。现在队列这种数据结构其实和链表很像，只是队列的操作有限制，因此**大锁这种方案我们跳过**。我们来看看 Michael 和 Scott 设计的更并发的队列：

```c
typedef struct __node_t
{
    int value;
    struct __node_t *next;
} node_t;

typedef struct queue_t
{
    node_t *head;
    node_t *tail;
    pthread_mutex_t headLock;
    pthread_mutex_t tailLock;
} queue_t;

void Queue_Init(queue_t *q)
{
    // tmp是假结点
    node_t *tmp = malloc(sizeof(node_t));
    tmp->next = NULL;
    q->head = q->tail = tmp;
    pthread_mutex_init(&q->headLock, NULL);
    pthread_mutex_init(&q->tailLock, NULL);
}

void Queue_Enqueue(queue_t *q, int value)
{
    node_t *tmp = malloc(sizeof(node_t));
    assert(tmp != NULL);
    tmp->value = value;
    tmp->next = NULL;

    pthread_mutex_lock(&q->tailLock);
    q->tail->next = tmp;
    q->tail = tmp;
    pthread_mutex_unlock(&q->tailLock);
}

int Queue_Dequeue(queue_t *q, int *value)
{
    pthread_mutex_lock(&q->headLock);
    node_t *tmp = q->head;
    node_t *newHead = tmp->next;
    // 如果是假结点，则视为空
    if (newHead == NULL)
    {
        pthread_mutex_unlock(&q->headLock);
        return -1; // queue was empty
    }
    *value = newHead->value;
    q->head = newHead;
    pthread_mutex_unlock(&q->headLock);
    free(tmp);
    return 0;
}
```

这段代码十分巧妙，用到了两个锁和一个哨兵节点。一个锁负责队头，另一个负责队尾，使得入队出队操作可以并发执行。而哨兵节点是在初始化的时候分配的，利用它分开了头和尾的操作。

### 并发散列表

最后讨论的是一个广泛应用的数据结构——散列表，为了突出重点，这里不对扩缩容做深入：

```c
#define BUCKETS (101)

typedef struct hash_t
{
    list_t lists[BUCKETS];
} hash_t;

void Hash_Init(hash_t *H)
{
    int i;
    for (i = 0; i < BUCKETS; i++)
    {
        List_Init(&H->lists[i]);
    }
}

int Hash_Insert(hash_t *H, int key)
{
    int bucket = key % BUCKETS;
    return List_Insert(&H->lists[bucket], key);
}

int Hash_Lookup(hash_t *H, int key)
{
    int bucket = key % BUCKETS;
    return List_Lookup(&H->lists[bucket], key);
}
```

这个散列表使用我们上面实现的**并发链表**，性能特别好。每个散列桶（每个桶都是一个链表）都有一个锁，而不是整个散列表只有一个锁，从而支持许多并发操作，其实就是**分段加锁**了。

这个简单的并发散列表扩展性（性能）极好，相较于单纯的链表。原因就是缩小了临界区，减少了不同线程间操作的冲突。



### 小结

并发其实并不意味着高性能，在实现并发数据结构时，先从最简单的开始（也就是加一把大锁），有性能问题的时候再做优化。关于最后一点，避免**不成熟的优化**（premature optimization），对于所有关心性能的开发者都有用。**我们让整个应用的某一小部分变快，却没有提高整体性能，其实没有价值**















































