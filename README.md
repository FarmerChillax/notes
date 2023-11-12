# 操作系统介绍

> 本系列文章将按照《Operating Systems: Three Easy Pieces》一书的章节顺序编写，结合原文与自己的感悟，以作笔记之用，如有不足之处，恳请在评论区指出



操作系统主要的三个部分分别是：虚拟化（virtualization）、并发（concurrency）和持久化（persistence），这是本书主要学习的3个关键概念。通过学习这三个概念来理解操作系统这门课程。

国内许多教材/八股文对操作系统的描述可能与本书差异较大，但我个人感觉其实都是描述一个东西，相比之下教材上只是描述的更具体。比如进程/线程其实是操作系统实现虚拟化的其中一个手段的等等。

### 虚拟化

#### CPU 虚拟化

下面是一个简单的程序，它所做的只是循环每秒打印出用户在命令行中传入的字符串：

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>  
#include <assert.h>
#include "common.h"

int main(int argc, char const *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "usage: cpu <string>\n");
        exit(1);
    }
    char *str = argv[1];
    while (1)
    {
        Spin(1);
        printf("%s\n", str);
    }
    

    return 0;
}
```

> `common.h` 头文件将放在本文末尾，当然你也可以访问[官网](https://pages.cs.wisc.edu/\~remzi/OSTEP/)/[GitHub](https://github.com/FarmerChillax/ostep-code/blob/master/intro/common.h)来获取

当我们在一个单处理器（cpu）的系统上编译并运行它，我们将看到以下内容：

```bash
farmer:~/studys/operating-systems/ch02$ ./cpu A
A
A
A
A
A
^C
farmer:~/studys/operating-systems/ch02$
```

这个结果看起来十分合理不是吗？现在让我们运行同一个程序的不同实例，来看看结果：

```
farmer:~/studys/operating-systems/ch02$ ./cpu A & ./cpu B & ./cpu C &
[1] 208278
[2] 208279
[3] 208280
B
C
A
B
C
C
B
A
B
A
C
B
A
C
...
```

现在事情开始有趣起来了，尽管只有一个处理器，但是这几个程序在我们用户的视角看来还是在同时运行的，看起来就像有多个处理器在同时执行这3个程序一样！

而这种将单个CPU（或其中一小部分）转换成看似多个CPU，从而让许多程序看似同时运行的技术，这就是所谓的虚拟化CPU（virtualizing the CPU）



#### 内存虚拟化

看完了CPU让我们来看看内存，现代机器提供的物理内存（physical memory）模型其实非常的简单，就是一个字节数组而已。程序通过指定一个地址（address）来访问、写入或更新存在那里的数据。

程序运行时一直要访问内存，程序将所有数据结构保存在内存中，并通过各种指令来访问它们，因此每次读取指令都会访问内存。下面来看一个简单的demo：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include "common.h"

int main(int argc, char const *argv[])
{
    int *p = malloc(sizeof(int)); //a1
    assert(p != NULL);
    printf("(%d) memory address of p: %08x\n", getpid(), (unsigned) p); //a2

    *p = 0; //a3
    while (1)
    {
        Spin(1);
        *p = *p + 1;
        printf("(%d) p: %d\n", getpid(), *p); // a4
    }

    return 0;
}
```

该程序的输出如下：

```bash
farmer:~/studys/operating-systems/ch02$ ./mem 
(208544) memory address of p: 18afa2a0
(208544) p: 1
(208544) p: 2
(208544) p: 3
(208544) p: 4
(208544) p: 5
^C
```

这个demo首先为 p 这个变量分配了一些内存（a1行）。然后打印出内存的地址（a2），然后将数字0放入新分配的内存（a3）的第一个空位中。最后程序循环，延迟一秒钟并递增 p 中保存的地址值。在每个打印语句中，它还会打印出正在运行程序的进程标识符（PID）。

同样的，我们再次运行多个实例来看看会发生什么：

```bash
farmer:~/studys/operating-systems/ch02$ ./mem & ./mem &
[1] 24113
[2] 24114
(24113) memory address of p: 00200000
(24114) memory address of p: 00200000
(24113) p: 1
(24114) p: 1
(24114) p: 2
(24113) p: 2
(24113) p: 3
(24114) p: 3
(24113) p: 4
(24114) p: 4
...
```

我们可以看到每个实例都在相同的地址（00200000）分配了内存，但每个似乎都独立更新了00200000处的值！就好像每个正在运行的程序都有自己的私有地址，而不是与其他正在运行的程序共享相同的物理内存。

而这真是操作系统虚拟化内存（virtualizing memory）时发生的情况，每个进程访问自己的私有虚拟地址空间（virtual address space）（有时称为地址空间，address space），操作系统以某种方式映射到机器的物理内存上，让正在运行的实例完全拥有自己的物理内存。

但实际情况是，物理内存是由操作系统管理的共享资源。而这也正是上文提到的「虚拟化」的内容。

### 并发

并发是这本书的第二个部分，并发只是一个用来代指「**同时处理很多事情**」所带来一系列问题的代词，而这些问题是在同时（并发）处理很多事情时出现且必须解决的。

那么为什么并发通常出现在《操作系统》这门课中呢？其实是因为并发问题首先出现在操作系统中，而随着软件工程的发展，现代多线程（multi-threaded）程序也存在相同的问题。我们来看一个具体的多线程例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include "common.h"
#include "common_threads.h"

volatile int counter = 0;
int loops;

void *worker(void *arg) {
    int i;
    for ( i = 0; i < loops; i++)
    {
        counter++;
    }
    return NULL;
}

int main(int argc, char *argv[]) {
    if (argc != 2) { 
	fprintf(stderr, "usage: threads <loops>\n"); 
	exit(1); 
    } 
    loops = atoi(argv[1]);
    pthread_t p1, p2;
    printf("Initial value : %d\n", counter);
    Pthread_create(&p1, NULL, worker, NULL); 
    Pthread_create(&p2, NULL, worker, NULL);
    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);
    printf("Final value   : %d\n", counter);
    return 0;
}
```

这个例子中利用`Pthread_create()`创建了两个线程（thread），每个线程开始在一个名为`worker()`的函数中运行，该函数中只是递增一个计数器，循环 loops 次。

下面是将变量loops的输入值设置为 1000 时的输出结果，根据代码我们可以很容易的猜到运行结果是 2000，因为每个线程会循环 1000 次并对计数器做一个累加的操作。也就是说当输入为 N 时，直观的预计结果为 2N：

```bash
farmer:~/studys/operating-systems/ch02$ ./thread 1000
Initial value : 0
Final value   : 2000
```

但如果对并发编程有点了解的话会发现这段代码中存在一个问题：计数器累加操作并非原子方式（atomically）的。让我们运行相同的程序，但 loops 的值更高，然后看看会发生什么：

```bash
farmer:~/studys/operating-systems/ch02$ ./thread 100000
Initial value : 0
Final value   : 100000
farmer:~/studys/operating-systems/ch02$ ./thread 100000
Initial value : 0
Final value   : 105018
farmer:~/studys/operating-systems/ch02$ ./thread 100000
Initial value : 0
Final value   : 135669
```

事实证明当我们将 loops 值设置的更高后，得到的最终值不是 200000。并且其最终值在每次运行中都是不一样的结果！

这些奇怪的，不合常理的结果与指令如何执行有关。上面程序中的关键部分是增加共享计数器（counter）的地方，它需要 3 条指令：一是将计数器的值翀内存中加载到寄存器，二是将其做递增操作，三是将其保存回内存。因为这三条指令并不是以原子的方式执行（所有的指令一次性执行）的，所以才会导致这些奇怪的事情发生。而这种问题通常叫：并发（concurrency）问题

### 持久性

我们都知道程序是运行在内存中的，如果发生断电或系统崩溃等情况，那么内存中的数据是容易丢失的。因为像 DRAM 这样的设备以易失（volatile）的方式存储数据，因此我们需要硬件和软件来持久化（persistently）的存储数据。这样的存储对于所有系统来说都十分重要，因为数据是无价的。

操作系统中管理硬盘的软件通常称为文件系统（file system）。因此它负责以可靠和高效的方式，将用户创建的任何文件（file）存储在系统的磁盘上。而不像 CPU 和内存一样需要操作系统提虚拟化，因为它是可以被多个程序所共有的。

```c
#include <stdio.h>
#include <unistd.h>
#include <assert.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <string.h>

int main(int argc, char *argv[]) {
    int fd = open("/tmp/file", O_WRONLY | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR);
    assert(fd >= 0);
    char buffer[20];
    sprintf(buffer, "hello world\n");
    int rc = write(fd, buffer, strlen(buffer));
    assert(rc == (strlen(buffer)));
    fsync(fd);
    close(fd);
    return 0;
}
```

上面的程序向操作系统发出了 3 个调用。一是 `open()` 调用，用来创建并打开一个文件。第二个是 `write()` 调用，将一些数据(「hello world\n」这串字符) 写入文件。第三则是 `close()` 调用来关闭文件，从而表明程序不会再向该文件写入更多的数据。

而上面提到的这些系统调用（system call）会被转到称为文件系统的操作系统部分，然后由文件系统处理这些请求，并向用户返回某种代码来表示结果。

首先确定新数据将驻留在磁盘上的哪个位置，然后在文件系统所维护的各种结构中对其进行记录。这样做需要向底层存储设备发出 I/O 请求，以读取现有结构或更新（写入）它们。所有写过设备驱动程序（device driver）的人都知道，让设备现表你执行某项操作是一个复杂而详细的过程。它需要深入了解低级别设备接口及其确切的语义。幸运的是，操作系统提供了一种通过系统调用来访问设备的标准和简单的方法。因此，OS 有时被视为标准库（standard library）。

出于性能方面的原因，大多数文件系统首先会`延迟`这些写操作一段时间，希望将其`批量分组`为较大的组。为了处理写入期间系统崩溃的问题，大多数文件系统都包含某种复杂的写入协议，如日志（journaling）或写时复制（copy-on-write），仔细`排序`写入磁盘的操作，以确保如果在写入序列期间发生故障，系统可以在之后恢复到合理的状态。为了使不同的通用操作更高效，文件系统采用了许多不同的数据结构和访问方法，从简单的列表到复杂的 `B 树`。

### 设计目标

设计目标指的是「开发和设计操作系统时所设定的主要目标和原则」，或者说是操作系统的一些基本设计原则。

1. **一个最基本的目标，是建立一些抽象（abstraction）**，让系统方便和易于使用。抽象对我们在计算机科学中做的每件事都很有帮助。抽象使得编写一个大型程序成为可能，将其划分为小而且容易理解的部分
2. **设计和实现操作系统的第二个目标，是提供高性能（performance）**。换言之，我们的目标是`最小化`操作系统的`开销`（minimize the overhead）。但是虚拟化的设计是为了易于使用，无形之中会增大开销，比如虚拟页的切换，cpu 的调度等等，所以尽可能的保持易用性与性能的平衡至关重要
3. **在应用程序之间以及在 OS 和应用程序之间提供保护（protection）**。因为我们希望让许多程序同时运行，所以要确保一个程序的恶意或偶然的不良行为不会损害其他程序。保护是操作系统基本原理之一的核心，这就是`隔离`（isolation）。让进程彼此隔离是保护的关键，因此决定了 OS 必须执行的大部分任务
4. **操作系统往往力求提供高度的可靠性（reliability）**。因为操作系统必须不间断运行，当它失效时，系统上运行的所有应用程序也会失效。



### 总结

本章节主要用于对操作系统全貌的一个介绍，回答了：“操作系统是什么” 的问题。

































