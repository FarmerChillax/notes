# 第三十一章-信号量

从前两篇文章可知**条件变量**必须和**锁**配合使用，那为什么不直接封装在一起呢？于是就有个信号量。

**信号量**只是将锁和单值条件的条件变量**封装**在一起，所以它不是一个全新的概念，它能实现的事锁加条件变量都能实现。对于比较复杂情况下的条件判断无法使用信号量解决，因为其只内置了一个简单的整型的 value 条件。

### 信号量的定义

信号量是有一个整数值的对象，可以用两个函数来操作它。在 POSIX 标准中，是`sem_wait()`和 `sem_post()`。

```c
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);
```

sem\_init 用于初始化信号量:

```c
int sem_wait(sem_t *s) {
    decrement the value of semaphore s by one
    wait if value of semaphore s is negative
}

int sem_post(sem_t *s) {
    increment the value of semaphore s by one
    if there are one or more threads waiting, wake one
}
```

* sem\_wait()要么`立刻返回`（调用 sem\_wait()时，信号量的值`大于等于 1`），同时`信号量减1`，要么会让调用线程`挂起`，直到之后的一个 post 操作。
* sem\_post()并没有等待某些条件满足。它直接`增加信号量`的值，如果有等待线程，`唤醒`其中一个。
* 当信号量的值为`负数`时，这个值就是`等待线程`的个数

### 二值信号量（锁）





























