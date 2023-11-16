# 第二十七章-插叙：线程 API

### 线程创建

```bash
#include <pthread.h>
int
pthread_create(
    pthread_t * thread,
    const pthread_attr_t * attr,
    void * (*start_routine)(void*),
    void * arg
);
```

上面的代码是 POSIX 中 创建线程的接口定义，该函数有 4 个参数，具体描述如下：

1. **thread** 是指向 pthread\_t 结构体类型的指针，将利用这个结构来与该线程进行交互，因此需要将其传入 `pthread_create()` 以便初始化，相当于线程的唯一标识（身份证）
2. **attr** 是用于指定该线程可能具有的属性。比如栈的大小，或者线程调度优先级等信息。
3. **\*start\_routine** 是一个函数指针，指向要运行的函数
4. **arg** 要运行的函数的参数

### 线程完成

通过`pthread_join`阻塞等待线程完成

```c
// 创建进程
pthread_t p;
pthread_create(&p, NULL, mythread, (void *) 100);、
// 等待进程完成
pthread_join(p, (void **) &m);
```

1. 第一个参数是 pthread\_t 类型，用于指定要等待的线程
2. 第二个参数是一个指针，用于获取返回值。因为函数可以返回任何东西，所以它被定义为返回一个指向 void 的指针。

### 锁

除了创建和等待线程之外，POSIX 线程库提供的最有用的函数集，可能就是通过锁（lock）来提供互斥的那些函数了。最基本的一对函数是：

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

使用的代码大概是这样子：

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
// 或是int rc = pthread_mutex_init(&lock, NULL);
pthread_mutex_lock(&lock);
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```

> 注意锁必须正确初始化，上面的代码展示了两种上锁的方法（使用 PTHREAD\_MUTEX\_INITIALIZER 常量或者使用 pthread\_mutex\_init() 函数，一般情况下更推荐用第二种。



### 条件变量（condition variable）

所有线程库还有一个主要组件，就是**条件变量**。当线程之间必须发生某种信号时，条件变量就很有用。比如一个线程在**等待**另一个线程继续**执行**某些操作。

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```

要使用条件变量，必须另外有一个与此条件**相关的锁**。在调用上述任何一个函数时，应该持有这个锁。

第一个函数`pthread_cond_wait()`使调用线程进入`休眠`状态，因此等待其他线程发出信号，通常当程序中的某些内容发生变化时，现在正在休眠的线程可能会关心它。典型的用法如下所示：

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_lock(&lock);
while (ready == 0)
    pthread_cond_wait(&cond, &lock);
pthread_mutex_unlock(&lock);
```

`唤醒线程`的代码运行在另外某个线程中，调用`pthread_cond_signal`时也需要持有`对应锁`。像下面这样：

```c
pthread_mutex_lock(&lock);
ready = 1;
pthread_cond_signal(&cond);
pthread_mutex_unlock(&lock);
```

`pthread_cond_wait`有第二个参数，因为它会`隐式`地`释放锁`，以便在其线程休眠后唤醒线程可以获取锁，之后又会**重新获得锁**。\
本例通过 while 判断 ready 的值的变更，而不是通过条件变量唤醒判断 ready 已变更。将唤醒视为某种事物可能已经发生变化的暗示，而不是绝对的事实，这样更安全

### 编译和运行

代码需要包括头文件 pthread.h 才能编译。链接时需要 pthread 库，增加 `-pthread` 标记。

```bash
prompt> gcc -o main main.c -Wall -pthread
```

### 小结

本章介绍了基本的 pthread 库，包括**线程创建**、**锁实现互斥执行、通过条件变量的信号和等待**。如需更多信息，请在 Linux 系统上输入 `man -k pthread` 来获取。

当你使用 POSIX 线程库（或者实际上，任何线程库）来构建多线程程序时，需要记住一些小而重 要的事情：

* **保持简洁**。最重要的一点，线程之间的锁和信号的代码应该尽可能简洁。复杂的线程交互容易产生缺陷。
* **让线程交互减到最少**。尽量减少线程之间的交互。每次交互都应该想清楚，并用验证过的、正确的方法来实现（很多方法会在后续章节中学习）。
* **初始化锁和条件变量**。未初始化的代码有时工作正常，有时失败，会产生奇怪的结果。
* **检查返回值**。当然，任何 C 和 UNIX 的程序，都应该检查返回值，这里也是一样。否则会导致古怪而难以理解的行为，让你尖叫，或者痛苦地揪自己的头发。
* **注意传给线程的参数和返回值**。具体来说，如果传递在栈上分配的变量的引用，可能就是在犯错误。
* **每个线程都有自己的栈**。类似于上一条，记住每一个线程都有自己的栈。因此，线程局部变量应该是线程私有的，其他线程不应该访问。线程之间共享数据，值要在堆（heap）或者其他全局可访问的位置。
* **线程之间总是通过条件变量发送信号**。切记不要用标记变量来同步。
* **多查手册**。尤其是 Linux 的 pthread 手册，有更多的细节、更丰富的内容。请仔细阅读！















