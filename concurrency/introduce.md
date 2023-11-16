# 第二十六章-并发介绍

前面我们已经看到了操作系统是如何将一个物理 CPU 编程多个虚拟 CPU，从而支持多个程序同时运行的假象；还看到了如何为每个进程创建巨大、私有的虚拟内存，让每个程序好像拥有自己的内存，而实际操作系统秘密的复用物理内存。

OSTEP 的第二部分将介绍并发相关的内容，因此本章将首先介绍为单个运行进程提供的新抽象：进程（thread）。经典的观点是一个程序只有一个执行点（一个程序计数器，用来存放要执行的指令），但多线程（multi- threaded）程序会有多个执行点（也就是多个程序计数器，每个都用于取指令与执行）。换个角度来看，每个线程类似于独立的进程，只有一点区别：它们共享地址空间，从而能够访问相同的数据。

### 线程的结构

单个线程的状态与进程状态非常类似。线程有一个**程序计数器**（Program Counter），记录程序从哪里获取指令。`每个线程`有自己的`一组`用于计算的`寄存器`。所以，如果有两个线程运行在一个处理器上，从运行一个线程（T1）`切换`到另一个线程（T2）时，必定发生**上下文切换**（context switch）。线程之间的上下文切换类似于进程间的上下文切换。对于进程，我们将状态保存到**进程控制块**（Process Control Block，PCB）。现在，我们需要一个或**多个线程控制块**（Thread Control Block，TCB），保存每个线程的状态。但是，与进程相比，线程之间的**上下文切换**有一点主要区别：**地址空间保持不变**（即不需要切换当前使用的**页表**）。

线程和进程之间另一个主要的区别在于栈。在传统的进程内存模型中，只有一个栈，通常位于地址空间的底部。而多线程的进程中，每个线程独立运行，因此地址空间中不只有一个栈，而是每个线程都有一个栈。如下图所示：

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="563"><figcaption></figcaption></figure>



### 实例：进程创建

下面代码演示了一个简单的多线程程序：

```c
#include <stdio.h>
#include <assert.h>
#include <pthread.h>

void *mythread(void *arg) {
    printf("%s\n", (char *) arg);
    return NULL;
}

int
main(int argc, char *argv[]) {
    pthread_t p1, p2;
    int rc;
    printf("main: begin\n");
    rc = pthread_create(&p1, NULL, mythread, "A"); assert(rc == 0);
    rc = pthread_create(&p2, NULL, mythread, "B"); assert(rc == 0);
    // join waits for the threads to finish
    rc = pthread_join(p1, NULL); assert(rc == 0);
    rc = pthread_join(p2, NULL); assert(rc == 0);
    printf("main: end\n");
    return 0;
}
```

这个代码中，主程序创建了两个线程，分别执行 `mythread()` 函数，但传参不一样。一旦线程创建，可能会立即运行或者处于就绪状态等待执行（这取决于调度程序）。下面三个表格中分别列举了几种可能的运行顺序：

情况一：

<table data-full-width="false"><thead><tr><th>主程序</th><th>线程一</th><th>线程二</th></tr></thead><tbody><tr><td><ol><li>开始运行</li><li>打印 "main:begin"</li><li>创建线程 1</li><li>创建线程 2</li><li>等待线程 1</li></ol></td><td></td><td></td></tr><tr><td></td><td><ol><li>运行</li><li>打印 “A”</li><li>返回</li></ol></td><td></td></tr><tr><td>等待线程 2</td><td></td><td></td></tr><tr><td></td><td></td><td><ol><li>运行</li><li>打印 “B”</li><li>返回</li></ol></td></tr><tr><td>打印 "main:end"</td><td></td><td></td></tr></tbody></table>

情况二：

<table><thead><tr><th width="295">主程序</th><th>线程一</th><th>线程二</th></tr></thead><tbody><tr><td><ol><li>开始运行</li><li>打印 "main:begin"</li><li>创建线程 1</li><li>创建线程 2</li></ol></td><td></td><td></td></tr><tr><td></td><td></td><td><p></p><ol><li>运行</li><li>打印 “B”</li><li>返回</li></ol></td></tr><tr><td>等待线程 1</td><td></td><td></td></tr><tr><td></td><td><p></p><ol><li>运行</li><li>打印 “A”</li><li>返回</li></ol></td><td></td></tr><tr><td><ol><li>等待线程 2</li><li>打印 "main:end"</li></ol></td><td></td><td></td></tr></tbody></table>

情况三：

<table><thead><tr><th width="295">主程序</th><th>线程一</th><th>线程二</th></tr></thead><tbody><tr><td><ol><li>开始运行</li><li>打印 "main:begin"</li><li>创建线程 1</li><li>创建线程 2</li></ol></td><td></td><td></td></tr><tr><td></td><td><ol><li>运行</li><li>打印 “A”</li><li>返回</li></ol></td><td></td></tr><tr><td>创建线程 2</td><td></td><td></td></tr><tr><td></td><td></td><td><ol><li>运行</li><li>打印 “B”</li><li>返回</li></ol></td></tr><tr><td><ol><li>等待线程 1</li><li>等待线程 2</li><li>打印 "main:end"</li></ol></td><td></td><td></td></tr></tbody></table>

不难看出线程使得编程变得复杂：已经很难说出什么时候会运行了！



### 为什么更糟糕：共享数据

两个线程递增同一个数，每次运行最终结果都不一样？原因是共享数据未保证操作`原子性` ，因为代码中看似一行代码的「加加」，在真实执行时是由几个指令完成的，因此存在并发问题。











