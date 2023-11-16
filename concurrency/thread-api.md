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

