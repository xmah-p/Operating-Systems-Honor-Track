- [Operating Systems](#operating-systems)
  - [Four Fundamental OS Concepts](#four-fundamental-os-concepts)
  - [Abstraction](#abstraction)
    - [Threads](#threads)
      - [Concurrency](#concurrency)
    - [Files and I/O](#files-and-io)
      - [High-level File API: Streams](#high-level-file-api-streams)
      - [Low-level File API: File Descriptors](#low-level-file-api-file-descriptors)
      - [How and Why of High-level File I/O](#how-and-why-of-high-level-file-io)
      - [Process State for File Descriptors](#process-state-for-file-descriptors)
      - [Pitfalls with OS Abstractions](#pitfalls-with-os-abstractions)
    - [IPC, Pipes and Sockets](#ipc-pipes-and-sockets)
      - [Pipe](#pipe)
      - [Socket](#socket)
  - [Synchroization](#synchroization)
    - [Producer-Consumer with a Bounded Buffer](#producer-consumer-with-a-bounded-buffer)
    - [Too Much Milk](#too-much-milk)
    - [Lock Implementation](#lock-implementation)
    - [Lock Implementation with Atomic Operations](#lock-implementation-with-atomic-operations)
    - [Monitor](#monitor)
    - [Readers/Writers](#readerswriters)
    - [Construct Monitor from Semaphores](#construct-monitor-from-semaphores)
  - [Scheduling](#scheduling)
    - [Multi-Core Scheduling](#multi-core-scheduling)
    - [Real-time Scheduling](#real-time-scheduling)
    - [Ensuring Progress](#ensuring-progress)
    - [Case Study](#case-study)
    - [Choosing the Right Scheduler](#choosing-the-right-scheduler)
    - [Deadlock](#deadlock)
    - [ZygOS](#zygos)
    - [Tiresias](#tiresias)
    - [DRF](#drf)
    - [FairRide](#fairride)


# Operating Systems

操作系统：为应用程序提供硬件资源的 special layer

- 为复杂硬件设备提供一层方便的抽象
- 对共享资源的访问提供保护
- Security and authentication
- 逻辑实体的沟通

操作系统是裁判、幻术师、胶水。

## Four Fundamental OS Concepts

- Thread
- Address space (with translation)
- Process
- Dual mode operation / Protection

简单的保护机制：Base and Bound

- Base: 起始地址
- Bound: 边界

两种实现：

- 加载时计算物理地址（加载的实现比较复杂），运行时检查边界
- 运行时计算物理地址（性能 overhead，不过 B&B 很简单）

B&B 的优点：简单、进程间隔离、进程和 OS 隔离

B&B 的缺点：Internal fragmentation（每个进程 heap 和 stack 之间的内存浪费）、External fragmentation（进程之间）

从用户态切换到内核态：

- 系统调用
- 外部中断
- 内部中断



## Abstraction 

### Threads

Motivation: Multiple Thing At Once (MTAO)

- Multiprocessing: 多 CPU
- Multiprogramming: 多进程
- Multithreading: 多线程

Concurrency（并发）不是 parallelism（并行）, 每个任务并非 simultaneously 进行。

线程有三种状态：RUNNING, READY, BLOCKED (**正在等待 I/O**)

程序开始后，可以通过系统调用创建线程。

```c
#include <pthread.h>

// 创建线程执行 start_routine(arg)
// 返回：若成功则为 0，若出错则非零
int pthread_create(pthread_t* tid, const pthread_attr_t* attr, void* (*start_routine)(void*), void* arg);

// 返回：调用线程的 TID
pthread_t pthread_self(void);

// 无返回值
void pthread_exit(void* thread_return);

// 返回：若成功则为 0，若出错则非零
int pthread_join(pthread_t tid, void** thread_return);
```

当顶层的线程例程返回时，线程会隐式地调用 `pthread_exit` 而终止。

通过调用 `pthread_exit`，线程可以显式地终止。如果主线程调用 `pthread_exit`，它会等待所有对等线程终止，然后再终止主线程和整个进程。

可选参数 `thread_return` 指定了线程例程的返回值。

`pthread_join` 会阻塞，直到线程 `tid` 终止。线程例程的返回值被存放在 `*thread_return` 中。此后，线程 `tid` 的资源被回收。

线程状态：

- 所有同地址空间的线程共享的
  - 全局变量、堆
  - I/O 状态（文件描述符、网络连接）
- 线程的私有状态
  - 保存在 Thread Control Block (TCB) 中
  - CPU 寄存器（包括 PC）
  - 执行栈（execution stack）
    - 包含局部变量、参数、返回地址

#### Concurrency

Mutual exclusion: 保证同一时间点只有一个线程访问共享资源。

Critical sectoin: Mutually exclusive 的代码片段。

```c
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t* mutex, const pthread_mutexattr_t* attr);
int pthread_mutex_lock(pthread_mutex_t* mutex);
int pthread_mutex_unlock(pthread_mutex_t* mutex);

int common = 162;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* thread_fn(void* arg) {
    pthread_mutex_lock(&mutex);
    common++;
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

信号量：`P` 和 `V`

- 可以用于实现 mutual exclusion
- 也可以用于 threads 之间的 signaling（`join` 为 `P`，`exit` 为 `V`）

多进程编程：`fork` 和 `exec`

- 为什么线程 API 只有 `pthread_create`，进程却有 `fork` 和 `exec`？
  - 可以只 `fork` 不 `exec`，从而将父子进程的代码都放在同一个可执行文件
  - 便于控制子进程的状态（在 `exec` 前设定子进程状态）

线程和进程的选择：

- 线程性能更强：上下文切换开销小、内存占用小、创建销毁/开销小
- 进程保护性强：互相隔离且独立的地址空间
- 线程间共享数据和通信更方便

### Files and I/O

Unix/POSIX: 一切皆“文件”

- 磁盘上的文件
- 设备 (terminals, printers, etc.)
- 网络 sockets
- 本地的进程间通信 (pipes)

文件系统的抽象:

文件:

- Named collection of data
- 字节序列
- Metadata

目录:

- 包含文件和目录
- 路径唯一确定一个文件或目录（整个文件系统内可以有重名文件，只要它们的路径不一样）

每个进程有一个 current working directory，可以通过 `chdir` 系统调用改变。相对路径与 CWD 有关。

#### High-level File API: Streams

```c
#include <stdio.h>

// 流：FILE* 类型
FILE* fopen(const char* path, const char* mode);
// mode: r, w, a, r+, w+, a+; b for binary mode
int fclose(FILE* fp);

int fseek(FILE* fp, long offset, int whence);
// whence: SEEK_SET, SEEK_CUR, SEEK_END
long ftell(FILE* fp);
void rewind(FILE* fp);    // fseek(fp, 0, SEEK_SET)

// C 程序执行时，三个标准流是隐式打开的。它们可以重定向
FILE* stdin;  // 0
FILE* stdout; // 1
FILE* stderr; // 2

// 字符
int fputc(int c, FILE* fp);
int fputs(const char* s, FILE* fp);
int fgetc(FILE* fp);
char* fgets(char* s, int size, FILE* fp);
// 块
int fread(void* ptr, size_t size, size_t nmemb, FILE* fp);    // 读取 nmemb 个 size 大小的块到 ptr
int fwrite(const void* ptr, size_t size, size_t nmemb, FILE* fp);
// 格式化
int fprintf(FILE* fp, const char* format, ...);
int fscanf(FILE* fp, const char* format, ...);

// 记得错误处理
FILE* input = fopen("input.txt", "r");
if (input == NULL) {
    // Prints our string and error msg.
    perror(“Failed to open input file”);
}
```

流：字节序列及当前位置

#### Low-level File API: File Descriptors

Unix I/O 设计思想：

- 一切皆文件
- Open before use: 访问控制
- 面向字节
- kernel buffered reads and writes
  - 统一化流式和块式设备
  - 提高效率
- 显式 close

```c
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>

// 系统调用
// open 系统调用返回 open file descriptor，或 < 0 的错误码
int open(const char* path, int flags, mode_t mode);
int creat(const char* path, mode_t mode);    
int close(int fd);

STDIN_FILENO, STDOUT_FILENO, STDERR_FILENO    // macros of values 0, 1, 2
int fileno(FILE* fp);    // 从 FILE* 得到 file descriptor
FILE* fdopen(int fd, const char* mode);    // 从 file descriptor 到 FILE*

// 系统调用
ssize_t read(int fd, void* buf, size_t count);    // 返回读取的字节数，或 < 0 的错误码
ssize_t write(int fd, const void* buf, size_t count);
off_t lseek(int fd, off_t offset, int whence);    // 返回新的文件偏移量，或 < 0 的错误码
// lseek 和 high-level API 的文件位置无关
```

Kerner buffering: read/write 系统调用昂贵，kernel 会缓存数据，减少系统调用次数。

返回文件描述符，而非文件描述条目的指针，是为了隔离用户和内核、提供抽象、增强安全性。

#### How and Why of High-level File I/O

```c
// High level
size_t fread(void* ptr, size_t size, size_t nmemb, FILE* fp) {
    // Do some work like a normal fn

    // asm code ... syscall # into %eax
    // put args into regs
    // special trap instruction

    // Kernel: get args from regs; dispatch to sys fn; read data; return value

    // get return value from regs
    // Do some more work like a normal fn
}

// Low level
ssize_t read(int fd, void* buf, size_t count) {
    // asm code ... syscall # into %eax
    // put args into regs
    // special trap instruction

    // Kernel: get args from regs; dispatch to sys fn; read data; return value

    // get return value from regs
}
```

`FILE*` 包括：

- 文件描述符
- 缓冲区
- 锁
- ...

当调用 `fwrite` 时，数据先被写入 `FILE` 的缓冲区，缓冲区满或遇到特定字符时会 flush（即将数据写入文件描述符）。

```c
char x = 'c';
FILE* f1 = fopen("file.txt", "wb");
fwrite("b", sizeof(char), 1, f1);
// fflush(f1);
FILE* f2 = fopen("file.txt", "rb");
fread(&x, sizeof(char), 1, f2);
// 没有 fflush 时，f1 的缓冲区可能还没写入文件，因此 f2 读取立即遇到 EOF, x 保持 'c'
```

使用 high-level API 时需要注意缓冲区 flush 的问题，但 low-level API 不需要（`write` 系统调用会直接写入文件描述符）。

Buffer in userspace: 

- 系统调用开销很大：byte by byte 的读写，吞吐量仅为 10 MB/s，但 `fgetc` 可以匹配 SSD 的速度
- 系统调用功能简单
  - 没有“读直到换行”

#### Process State for File Descriptors

`open` 系统调用返回一个 file descriptor（`int`），并在内核创建一个 open file description，其中包含文件在磁盘上的位置、当前文件位置等。

`fork` 时，子进程会继承父进程的 file descriptor 和 open file description（后者被 aliased, 即共享, 并不创建新的 open file description）。当所有进程的 file descriptor 都被 `close` 时，open file description 才会被释放。

- 文件位置也会共享
- 但如果子进程调用 `open`，则会创建新的 open file description，文件位置不会和父进程共享。

这便于进程间的资源共享：

- 一切皆文件，这使得文件、网络连接、终端、管道等都可以在父子进程间共享

重定向：`dup` 和 `dup2`

- 创建一个新的 file descriptor，指向**同一个** open file description

#### Pitfalls with OS Abstractions

多线程进程的 `fork`：子进程**只继承调用 `fork` 时的线程，其他线程消失**。如果其他线程有锁或正在写入，可能导致麻烦。

在子进程调用 `exec` 是安全的，因为 `exec` 会覆写整个地址空间。

不要混用 high-level 和 low-level API，否则可能导致缓冲区不一致。

```c
char x[10];
char y[10];
FILE* f = fopen("foo.txt", "rb");
int fd = fileno(f);
fread(x, 10, 1, f); // read 10 bytes from f
read(fd, y, 10); // assumes that this returns data starting at offset 10

// 由于 fread 会读入一大块数据（可能是整个文件），y 读取的数据不会是文件的第 10 个到第 19 个字节
```

### IPC, Pipes and Sockets

#### Pipe

进程间通信：如果用文件，性能会太差。

Unix Pipe: **存储在内存中的固定大小队列**

- 对单向队列的抽象
- 本地 IPC
- 文件描述符通过继承获得

```c
#include <unistd.h>

int pipe(int pipefd[2]);
// 分配两个文件描述符，用于 IPC
// pipefd[0]: read end
// pipefd[1]: write end
// 如果 pipe 满了，write 会阻塞，直到有空间
// 如果 pipe 空了，read 会阻塞，直到有数据
```

```c
#include <unistd.h>

int pipefd[2];
if (pipe(pipefd) == -1) {
    fprintf(stderr, "Pipe failed\n");
    return EXIT_FAILURE;
}

// 自己给自己发消息
ssize_t writelen = write(pipefd[1], msg, strlen(msg) + 1);
ssize_t readlen = read(pipefd[0], buf, BUF_SIZE);
// 省略错误处理

// 父子进程通信: 父进程写，子进程读
pid_t pid = fork();
if (pid < 0) {
    fprintf(stderr, "Fork failed\n");
    return EXIT_FAILURE;
} else if (pid != 0) {
    // parent
    ssize_t writelen = write(pipefd[1], msg, strlen(msg) + 1);
    close(pipefd[0]);
} else {
    // child
    ssize_t readlen = read(pipefd[0], buf, BUF_SIZE);
    close(pipefd[1]);
}
```

当所有写端文件描述符关闭时，读端会检测到 EOF。

当所有读端文件描述符关闭时，写端调用 `write` 会生成 SIGPIPE 信号（如果忽略此信号，`write` 会 `EPIPE` 报错）。

Pipe 是单向的，父子进程必须关闭不用的文件描述符（**否则读取进程检测不到 EOF**），双向通信需要创建两个 pipe。

#### Socket

沟通需要协议：

- Syntax：信息的组织结构
- Semantics：信息的意义 

客户端-服务器通信：

- 服务器总是开机，处理来自多个客户端的请求
- 客户端有时开机，有时关机

（TCP）网络连接：**两个（不同机器的）进程间的双向字节流**。

- 包括一个从 Alice 到 Bob 的队列和一个从 Bob 到 Alice 的队列。
- `write` 向输出队列写入数据，`read` 从输入队列读取数据。

Socket：对于网络连接的一个 endpoint 的抽象，就像一个文件描述符。

- 对双向队列的抽象
- 远程 IPC
- 文件描述符通过 `socket`、`bind`、`listen`、`connect`、`accept` 获得

```c
// Echo client-server 
void client(int sockfd) {
    int n;
    char sndbuf[MAXIN];
    char rcvbuf[MAXOUT];
    while (true) {
        fgets(sndbuf, MAXIN, stdin);
        write(sockfd, sndbuf, strlen(sndbuf));
        memset(rcvbuf, 0, MAXOUT);
        n = read(sockfd, rcvbuf, MAXOUT);
        printf("Echo: %s", rcvbuf);
    }
}

void server(int sockfd) {
    int n;
    char buf[MAXLINE];
    while (true) {
        n = read(sockfd, buf, MAXLINE);
        if (n <= 0) return;
        write(sockfd, buf, n);
    }
}
```

以上代码预设了：

- Reliable: 没有数据丢失
- In-order: 数据按顺序到达

`read` 预设已经有数据可读，当没有数据可读时会阻塞。

Server socket：

- 有文件描述符
- 不能读或写
- 两种操作
  - `listen`：开始允许客户端连接
  - `accept`：接受连接请求，返回一个新的 socket（可以读写）

一个连接由一个 5-tuple 表示：(源 IP, 源端口, 目的 IP, 目的端口, 协议)

客户端协议：

```c
char* host_name;
char* port_name;

struct addrinfo* server = lookup(host_name, port_name);
int sockfd = socket(server->ai_family, server->ai_socktype, server->ai_protocol);

connect(sockfd, server->ai_addr, server->ai_addrlen);

run_client(sockfd);

close(sockfd);
```

服务器协议 v1：

```c
char* port_name;
struct addrinfo* server = setup_addrinfo(port_name);

int server_socket = socket(server->ai_family, server->ai_socktype, server->ai_protocol);

bind(server_socket, server->ai_addr, server->ai_addrlen);
listen(server_socket, MAX_QUEUE);

while (true) {
  int conn_socket = accept(server_socket, NULL, NULL);
  serve_client(conn_socket);
  close(conn_socket);
}
close(server_socket);
```

服务器协议 v2：为了保护服务器，为每个连接创建一个子进程。

```c
// ...
while (true) {
  int conn_socket = accept(server_socket, NULL, NULL);
  pid_t pid = fork();
  if (pid == 0) {
    // child process
    close(server_socket);
    serve_client(conn_socket);
    close(conn_socket);
    exit(0);
  } else if (pid > 0) {
    // parent process
    close(conn_socket);
    // wait(NULL);    // 不用等待子进程
  } else {
    // fork failed
    perror("fork failed");
  }
}
```

## Synchroization

**Process Control Block**: 内核将每个进程视为一个进程控制块（PCB），包含：

- 进程状态：RUNNING, READY, BLOCKED
- 寄存器
- PID、用户、优先级、...
- 执行时间
- 内存空间、内存翻译

内核的 scheduler 会根据进程状态和优先级决定下一个运行的进程（在 PCB 之间分配 CPU 资源），如果 ready queue 为空，则会选择 idle 进程。

Context switch: 由 interrupt 或系统调用触发，包括将状态保存到 PCB、...、从 PCB 恢复状态。

PCB 在各种队列之间移动。当它们没有运行时，它们便在 scheduler 的 ready 队列中。当它们请求 I/O 操作时，它们便在 I/O 设备的队列中。

一个进程的线程之间共享地址空间（堆、全局变量、代码段），每个线程有独立的 TCB。

The dispatch loop:

```c
Loop {
    RunThread();
    ChooseNextThread();
    SaveStateOfCPU(curTCB);
    LoadStateOfCPU(nextTCB);
}
```

`RunThread`：

- 加载线程的状态（寄存器、PC、栈指针）
- 加载环境（虚拟内存空间）
- 跳转到线程的入口点。

Dispatcher 拿回控制权：

- 内部事件：线程主动放弃 CPU
  - I/O blocking（`read` 系统调用 -> trap 到 OS -> 内核发射读取命令 -> switch）
  - 等待其他线程的“信号”（调用 `wait`）
  - 调用 `yield`
- 外部事件：中断使得 OS 拿回控制权
  - Interrupts
    - Timer：保证在没有内部事件时，每个线程也都有机会运行

`yield` 会 trap 到 OS，调度器会选择下一个线程。

进程对比线程：

- Switch overhead：线程小，进程大
- Protection：同进程弱（共享地址空间）
- Sharing overhead：同进程小，不同进程大
- Parallelism：进程不能多核并行，线程可以

TCB 和线程栈的初始化：

- 初始化 TCB
  - 栈指针：指向栈顶
  - PC 返回地址：OS 汇编 routine `ThreadRoot`
  - 寄存器 `a0`、`a1` 初始化为 `fcnPtr` 和 `fcnArgPtr`

线程的开始：

- `switch` 会选择新线程的 TCB，返回到其 `ThreadRoot` 的起始位置，开始运行新线程
- `ThreadRoot` 会：
  - 做一些初始化工作，如记录线程起始时间
  - 切换到用户模式
  - 调用 `fcnPtr(fcnArgPtr)`
  - 清理线程资源，唤醒睡眠线程

Shinjuku：

- 多核、小任务，进程切换的 OS 开销太大。
- 此问题的常见解决方案：
  - OS bypass
  - Polling (无中断，避免中断开销)
  - Run-to-completion (无调度，避免上下文切换)
  - d-FSFS (distributed queues + First Come First Serve scheduling)：网卡通过 Receive Side Scaling 把请求随机发给多个 CPU 中的某一个，每个 CPU 有自己的队列，队列中的任务按 FCFS 运行
- d-FCFS 的问题：Queue imbalance，某几个有难，其他的围观（non work-conserving）
- 解决方案：中心化的队列，c-FCFS
- 问题：如果所有核都在忙长请求，新的短请求会被这些长请求阻塞，造成 long tail latency。
- Shinjuku：使用类似 Linux 的抢占式调度
  - 如果采用 1ms 的时间片，对于给定的 Bimodal 请求来说永远不会触发调度，表现和 c-FCFS 完全一样
  - 如果采用 5us 的时间片，表现接近最优
  - Shinjuku 的贡献：实现了 us 级别的上下文切换
- 方法：
  - Dedicated core for scheduling and queue management
  - Leverage hardware support for virtualization for fast preemption
  - Very fast context switching in user space
  - Match scheduling policy to task distribution and target latency

相比于 Event Driven 编程，多线程可以更简单地实现 **I/O 和计算的 overlap**（前者需要 deconstruct code，代码复杂度高）。

原子操作：大部分机器中，从内存读取和赋值是原子的。锁可以用于保证原子性。


### Producer-Consumer with a Bounded Buffer

问题：生产者和消费者共享一个有限大小的缓冲区。生产者往缓冲区中放入数据，消费者从缓冲区中取出数据。

```c
Producer(item) {
    acquire(&buf_lock);
    while (buffer_full) {}
    enqueue(item);
    release(&buf_lock);
}

Consumer() {
    acquire(&buf_lock);
    while (buffer_empty) {}
    item = dequeue();
    release(&buf_lock);
}
```

问题：若缓冲区满，`Producer` 会陷入 `while` 死循环，而 `Consumer` 会在 `acquire` 时被阻塞，导致死锁。缓冲区空时也是如此。

简单粗暴的解决方案：

```c
Producer(item) {
    acquire(&buf_lock);
    while (buffer_full) {
        release(&buf_lock);
        acquire(&buf_lock);
    }
    enqueue(item);
    release(&buf_lock);
}

Consumer() {
    acquire(&buf_lock);
    while (buffer_empty) {
        release(&buf_lock);
        acquire(&buf_lock);
    }
    item = dequeue();
    release(&buf_lock);
}
```

问题：性能低下。若缓冲区满，需要恰好在 `while` 循环中 `release` 和 `acquire` 之间切换线程才能跳出死锁。缓冲区空时也是如此。

信号量（semaphore）：支持 `P` 和 `V` 操作的非负整数。

问题的正确性约束：

1. 若空，则消费者等待
2. 若满，则生产者等待
3. 生产者和消费者不能同时操作缓冲区（mutual exclusion）

通常，**每个约束对应一个信号量，信号量的值代表资源的可用数量**。

```c
Semaphore mutex = 1;    // 互斥锁
Semaphore empty = N;    // 生产者消耗一个空槽，释放一个满槽
Semaphore full = 0;     // 消费者消耗一个满槽，释放一个空槽

Producer(item) {
    P(&empty);
    P(&mutex);
    enqueue(item);
    V(&mutex);
    V(&full);
}

Consumer(&) {
    P(&full);
    P(&mutex);
    item = dequeue();
    V(&mutex);
    V(&empty);
}
```

**`P(&full)` 必须在 `P(&mutex)` 之前，否则可能导致死锁**。

`V(&empty)` 则不必在 `V(&mutex)` 之后（不过先 `V(&mutex)` 性能稍高）。

**即使我们有多个生产者和多个消费者**，以上代码也能正确工作。

### Too Much Milk

问题：多个舍友共享冰箱，冰箱空时每个人可以买牛奶放入。如果 Alice 正在买牛奶，Bob 查看冰箱为空，也去买牛奶，会导致牛奶买多。

可以在查看前加锁，放入牛奶后解锁。问题：如果 Bob 只是想拿橙汁，Alice 正在买牛奶，Bob 会被阻塞。

现在假设我们不会实现锁：

Solution 1

```c
if (noMilk) {
    if (noNote) {
        leave note;
        buy milk;
        remove note;
    }
}
```

问题：假如在 `if (noNote)` 和 `leave note` 之间切换，仍然会导致牛奶买多。

如果我们将 note 范围扩大：

```c
leave note;
if (noMilk) {
    if (noNote) {
        buy milk;
    }
}
remove note;
```

没人会买牛奶：会被自己的 note 阻塞。

Solution 2

```c
// Thread A
leave note A;
if (noNote B) {
    if (noMilk) {
        buy milk;
    }
}
remove note A;

// Thread B
leave note B;
if (noNote A) {
    if (noMilk) {
        buy milk;
    }
}
remove note B;
```

问题：**活锁**

- 线程 A 留下 note A 后切换
- 线程 B 留下 note B 后切换
- 则两个线程都不会买牛奶

Solution 3
    
```c
// Thread A
leave note A;
while (Note B) {}    // spin-wait
if (noMilk) {
    buy milk;
}
remove note A;

// Thread B
leave note B;
if (noNote A) {
    if (noMilk) {
        buy milk;
    }
}
remove note B;
```

线程 B 的实现没有变，线程 A 的逻辑从“如果有 note B 就不买牛奶”变成了“如果有 note B 就一直等”，因此**最终线程 A 一定会执行 `if` 语句**，解决了活锁的问题。

可以 work，但复杂，每个线程的代码都不同，难以支持多个线程。且包括 busy waiting。

需要硬件提供比 atomic load/store 更强的原子操作：通过锁。

Solution 4

```c
acquire(lock);
if (noMilk) {
    buy milk;
}
release(lock);
```

### Lock Implementation

Naive 锁实现：加锁禁用中断，解锁开启中断。

- dispatcher 通过内部事件和外部事件两种方式拿回控制权
- 对单核系统来说，只要不引发内部事件，并通过禁用中断避免外部事件，就可以防止切换
- 问题：
  - 不能为用户所使用：用户加锁后进入死循环，系统死机
  - 多核下无效：多核情况下，一个核禁用中断，另一个核无法禁用中断
  - 期间不能响应外部中断，破坏系统实时性
  - 不支持嵌套锁

更好的锁实现：

```c
int value = FREE;

acquire() {
    disable interrupts;
    if (value == BUSY) {
        // 如果在此处 enable interrupts
        // 则若在此处切换到 release 线程，则此时 wait queue 没有当前线程
        // 如果 ready queue 为空
        // 就会浪费一次调度
        put thread on wait queue
        // 如果在此处 enable interrupts
        // 则若在此处切换到 release 线程，当前线程会平白 sleep 一次
        go to sleep
        // 我们希望在此处 enable interrupts
        // 但这里已经进入 sleep 了
        // 我们只能令下一个切换到的线程 enable interrupts
    } else
        value = BUSY;
    enable interrupts;
}

release() {
    disable interrupts;
    if (wait queue is not empty) {
        remove thread from queue
        place thread on ready queue
    } else
        value = FREE;
    enable interrupts;
}
```

以上代码中包含检查并修改多个共享变量，因此需要禁用中断。（否则如果 `value` 值空闲，则两个线程会同时认为它们持有锁）

关键区域比 naive 实现短。

在 `sleep` 后重新开启中断：

- 每个线程在睡眠前禁用中断
- 每个线程在睡眠返回时首先开启中断

### Lock Implementation with Atomic Operations

禁用中断锁实现的问题：

- 不适用于用户模式
- 不适用于多核：多核禁用中断需要 message passing，耗时

原子的 read-modify-write 指令：

```c
// 多数架构都支持
test&set(&addr) {
    res = *addr;
    *addr = 1;
    return res;
}

// x86
swap(&addr, reg) {
    res = *addr;
    *addr = reg;
    return res;
}

// 68000
compare&swap(&addr, reg1, reg2) {
    if (reg1 == *addr) {
        *addr = reg2;
        return success;
    } else {
        return failure;
    }
}
```

naive 的利用 `test&set` 的锁实现：

```c
int val = 0;    // 0: free, 1: busy

acquire() {
    while (test&set(val)) {}
    // 如果 val == 0, 则 test&set 返回 0, val 被设为 1, 不会进入循环
    // 如果 val == 1, 则 test&set 返回 1, val 被设为 1, busy waiting
}

release() {
    val = 0;
}
```

问题：

- **busy waiting**。对多核系统来说，每一次 `test&set` 都是一次写，值会在 cache 间 ping-pong，性能很差。
- Priority inversion：低优先级线程持有锁，高优先级线程无法获得锁（不会被抢占）。

优点：

- 不用禁用中断
- 用户模式可用
- 多核可用

改进：

```c
int guard = 0;
int val = FREE;

acquire() {
    while (test&set(guard)) {}
    if (val == BUSY) {
        put thread on wait queue
        go to sleep & guard = 0
    } else {
        val = BUSY;
        guard = 0;
    }
}

release() {
    while (test&set(guard)) {}
    if (wait queue is not empty) {
        remove thread from queue
        place thread on ready queue
    } else {
        val = FREE;
    }
    guard = 0;
}

// guard 就是之前的 naive 锁，我们用小锁保护了大锁
```

优点：

- 小锁保护大锁，显著减少了临界区的范围（仅包括检查 `val` 和操作等待队列）
- 主锁会挂起并让出 CPU

相比于之前禁用中断的锁实现，我们把：

- 禁用中断替换为 `while (test&set(guard)) {}`
- `enable interrupts` 替换为 `guard = 0`

### Monitor

信号量有双重用途：互斥和调度约束。调转 `P` 的顺序可能导致死锁，这很不显然。

更干净的方法：用 Lock 实现 mutual exclusion，用 Condition Variable 实现 scheduling constraints。

Monitor：一个 Lock 以及零个或多个 Condition Variables。

Conditional Variable：临界区内等待某个特定条件的线程的队列。

Operations: 必须在获得锁的情况下调用

- `wait(&lock)`: 原子地释放锁并 sleep。在被唤醒后返回前，重新获得锁。
- `signal()`: 唤醒一个等待的线程。
- `broadcast()`: 唤醒所有等待的线程。

信号量不能在临界区内 sleep，而 Condition Variable 可以。

无限 buffer 的生产者消费者问题：

```c
lock buf_lock;
cond buf_CV;
queue queue;

Producer(item) {
    acquire(&buf_lock);
    enqueue(item);
    signal(&buf_CV);
    release(&buf_lock);
}

Consumer() {
    acquire(&buf_lock);
    // 在 Hoare scheduling 中，此处应为 if
    // 因为 signal 会将锁和 CPU 交给等待的线程，等待的线程立即开始运行
    // 出了临界区/又 wait 之后，锁和 CPU 还给 signal 的线程
    // 这不容易实现，而且性能一般
    while (queue is empty) {
        wait(&buf_CV, &buf_lock);
    }
    // 大部分 OS 采用 Mesa scheduling，因此此处需要用 while
    // signal 只会将等待的线程加入 ready queue，而不是立即执行上下文切换
    // 因此等真的切换到等待的线程时，可能条件已经不满足：需要重新检查
    // 例如：消费者 A 发现队列为空，开始等待。生产者 B 生产了一个 item，唤醒了 A
    // 但由于 A 不是立即被调度的，如果 A 被调度前队列又变成空的了，则 A 就必须再次等待
    item = dequeue();
    release(&buf_lock);
    return item;
}
```

有限 buffer 的生产者消费者问题：

```c
lock buf_lock;
cond consumer_CV, producer_CV;

Producer(item) {
    acquire(&buf_lock);
    while (queue is full) {
        wait(&consumer_CV, &buf_lock);    // 唤醒一个消费者
    }
    enqueue(item);
    signal(&producer_CV);
    release(&buf_lock);
}

Consumer() {
    acquire(&buf_lock);
    while (queue is empty) {
        wait(&producer_CV, &buf_lock);    // 唤醒一个生产者
    }
    item = dequeue();
    signal(&consumer_CV);
    release(&buf_lock);
    return item;
}
```

### Readers/Writers

正确性约束：

- 多个读者可以同时读
- 一个写者写时，不能有读者读，也不能有其他写者写
- 每次只能有一个线程操作共享数据
- （写者优先）

基本实现：

状态变量（被 `lock` 保护）：

- `AR`: 活跃读者数量
- `WR`: 等待读者数量
- `AW`: 活跃写者数量
- `WW`: 等待写者数量
- `okToRead`
- `okToWrite`

```c
Reader() {
    // 等待写者完成
    acquire(&lock);
    while (AW + WW > 0) {          // 有写者（非活跃写者也要等），不需要等读者，因为多个读者可以同时读
        WR++;    // 我们开始等待
        wait(&okToRead, &lock);
        WR--;    // 我们等完了
    }
    AR++;        // 我们活跃了！
    release(&lock);    // 这里要释放锁，从而其他读者可以同时读

    read data

    acquire(&lock);
    AR--;       // 我们不活跃了
    if (AR == 0 && WW > 0)   // 注意：删去这个判断不影响正确性 只是会变慢
        signal(&okToWrite);    // 改成 broadcast 不影响正确性，但会变慢
    release(&lock);
}

Writer() {
    // 等待读者完成
    acquire(&lock);
    while (AR > 0 || AW > 0) {    // 有活跃读者或写者
        WW++;    // 我们开始等待
        wait(&okToWrite, &lock);
        WW--;    // 我们等完了
    }
    AW++;        // 我们活跃了！
    release(&lock);

    write data

    acquire(&lock);
    AW--;       // 我们不活跃了
    if (WW > 0) {
        signal(&okToWrite);
    } else if (WR > 0) {
        broadcast(&okToRead);    // 唤醒所有读者（多个读者可以同时读）
        // 这里不能用 signal
        // 因为读者只会唤醒写者，不会唤醒其他读者
        // 如果有一个写者和一百个读者，那换成 signal，就会导致 99 个读者饿死
    }
    release(&lock);
}
```

这是写者优先的实现，读者可能饥饿（一直有新写者），写者不会饥饿。

> 考虑读写者序列 R1, R2, W1, R3，假设读写操作用时很长，则我们的实现会出现 R1, R2 同时读，W1 在等 R1, R2 完成，R3 在等 W1 完成（而不是 R1, R2, R3 都在同时读）。

可以只用一个条件变量 `okContinue`，此时必须用 `broadcast`，而不是 `signal`。

```c
Reader() {
    acquire(&lock);
    while (AW + WW > 0) {
        WR++;  
        wait(&okContinue, &lock);
        WR--;   
    }
    AR++;   
    release(&lock);   

    read data

    acquire(&lock);
    AR--;      
    if (AR == 0 && WW > 0)  
        broadcast(&okContinue);   // 唯一区别是 signal 变为 broadcast
    release(&lock);
}

Writer() {
    acquire(&lock);
    while (AR > 0 || AW > 0) {    
        WW++;    
        wait(&okContinue, &lock);
        WW--;    
    }
    AW++;
    release(&lock);

    write data

    acquire(&lock);
    AW--;
    if (WW > 0 || WR > 0) 
        broadcast(&okContinue);    
    release(&lock);
}
```

假如直接把 `okToRead` 和 `okToWrite` 替换为 `okContinue`，则会导致死锁：

- 读写者序列 R1, W1, R2
- R1 正在读，W1 和 R2 等待
- R1 读完，唤醒了 R2
- 则 R2 在等 W1，W1 又在等 R2 的 signal，死锁

在完整的实现中，正确性可以保证：

- 我们有 `lock`，所以大家还是“一个一个出来”的
- 如果有在等待的写者，因为这是写者优先实现，`broadcast` 后某一个写者将变成活跃状态，且只有此写者活跃
- 否则 `broadcast` 后一个读者变成活跃状态，随即所有读者都变为活跃状态

读者优先？FIFO（且相邻读者并发读）？用锁的实现？信号量？

### Construct Monitor from Semaphores

```c
// naive
wait(Lock* lock, Sema* sem) {
    release(lock);
    P(sem);
    acquire(lock);
}

signal(Lock* lock, Sema* sem) {
    V(sem);
}
```

问题：条件变量没有历史，信号量有历史。

- 先 `signal`，再 `wait`，则 `wait` 应当阻塞
- 先 `V` ，后 `P`，则 `P` 不应当阻塞
- 条件变量和信号量语义不同

如果修改 `signal` 的实现：

```c
signal(Lock* lock, Sema* sem) {
    if semaphore queue is not empty
        V(sem);
}
```

不再有上述问题，但有竞态条件：在 `wait` 内 `release` 之后切换至 `signal`，则这个 `signal` 会被丢弃，而不是唤醒 `wait` 的线程（尽管它本该唤醒），导致 `wait` 阻塞。

此外，检查信号量队列是否为空本身是非法操作。

(OSDI 06) The Chubby lock service for loosely-coupled distributed systems

## Scheduling

任务：

- 一个系统被不同 user 使用，每个 user 有若干 program，每个 program 有若干 thread。
- 如何调度？
  - 以某些特定指标为目标，优化 CPU 时间的分配
- 如何保证公平性？
  - 在 user 层面公平还是在 program 层面公平？
  - 我开 1 个程序，你开 100 个程序，多数系统中后者 CPU 时间更多

执行模型：程序在 CPU burst 和 I/O burst 之间不断切换。调度器需要将 CPU 时间分配给即将 CPU burst 的线程，从而最大化 CPU 利用率。

调度器的目标：

- 最小化**完成时间（completion time）**
  - 定义：**任务下达到结束的延迟**
  - 快速响应
  - **等待时间（waiting time）**：任务**在 ready queue 中的时间**
- 最大化**吞吐量（throughput）**
  - 减少开销（如上下文切换）
  - 高效利用资源（CPU、硬盘、内存）
- 公平
- （对于实时系统）可预测性

**First-Come, First-Served (FCFS) Scheduling**：

- 每个进程按照到达顺序排队
- 优点：简单。缓存友好。
- 问题：Head-of-line blocking，先到的长任务会阻塞后到的短任务。对短任务的响应时间经常很差。
- 性能与任务到达顺序有关。短任务先到的话，响应时间表现更好。

**Round Robin (RR) Scheduling**：

- 每个进程分配一个时间片，时间片用完后被**抢占**，放入 ready queue 的末尾
- 时间片（time quantum）
  - 如果太大，退化为 FCFS，**等待时间增加**
  - 如果太小，上下文切换频繁，overhead 大，**吞吐量减小**。
    - **即使上下文切换开销为零，也会导致完成时间增加**
    - 相比集中完成，雨露均沾会导致大多数任务都在整个过程的后半段完成
  - 必须相对上下文切换开销来说足够大
- 所有任务的都不会等待超过 $(n-1)q$ 的时间
- 优点：在等待时间方面公平，**对短任务友好**
- 问题：对长任务来说，上下文切换开销累积。缓存不友好。
- 典型的时间片大小：10ms 到 100ms

如果任务长度均匀，且都比较长，FCFS 更好（无上下文切换开销）。反之，如果任务长度不均匀，RR 更好。

**严格优先级调度（Strict Priority Scheduling）**：

- 总是先执行优先级高的任务
- 每个优先级队列是按照 RR 执行的
- 问题：
  - Starvation：低优先级的任务被阻塞
    - 不公平（公平与平均完成时间的 trade-off）
  - Priority inversion：低优先级的任务持有锁，高优先级的任务请求此锁被阻塞，而中优先级的任务妨碍低优先级任务释放锁
    - 解决方法：优先级捐献
- 公平性的实现：
  - 每个 queue 分享 CPU 时间的特定份额
    - 高速通道人太多，反而比低速通道慢
  - 提高未被执行的任务的优先级
    - 所有任务都提升 => 互动性任务响应时间变差


假如我们能预测未来，就可以仿照最优 FCFS 调度：

- **Shortest Job First (SJF)**：
  - 每次都选择下一个 CPU burst 最短的任务
  - 也叫 **Shortest Time to Completion First (STCF)**。
  - 对**平均完成时间**来说，是**最优的非抢占式调度**
- **Shortest Remaining Time First (SRTF)**：
  - SJF 的**抢占式版本**
  - **如果新任务的完成时间比当前任务的剩余完成时间短，抢占当前任务**
  - 又叫 **Shortest Remaining Time to Completion First (SRTCF)**。
  - 对**平均完成时间**来说，是**最优的抢占式调度**。比 SJF 更好。
- 相比 RR，SRTF 可以在保证最小化平均完成时间的同时减少上下文切换次数。
- 优点：**最优的平均完成时间**
- 缺点：
  - 如果短任务太多，可能饿死长任务（**不公平**）
  - 需要预测未来：**我怎么知道这个任务要跑多久？**
    - 适应性：根据历史数据来决定政策。$\hat{t}_n = f(t_{n-1}, t_{n-2}, \ldots)$

**Lottery Scheduling**：

- 每个任务分配一定数量的 lottery tickets
  - 短的更多，长的更少，从而模拟 SRTF
  - 每个任务至少获得一个 ticket，以防止饥饿
- 每个时间片随机选择一个 ticket，执行中奖的任务
- 平均而言，任务获得的 CPU 时间将和其分得的彩票数量成正比
- 相比严格优先级调度的优点：对负载变化的反应更柔和

**Multi-Level Feedback Scheduling**：

- 多个队列，每个队列有不同的优先级
- 队列之间的调度：
  - Fixed priority：先高优先级队列，后低优先级队列
  - Time slice：每个队列分享特定份额的 CPU 时间（例如最高优先级 70%，次高优先级 20%，最低优先级 10%）
- 每个队列内的调度：
  - 前台 RR，后台 FCFS，
  - 前台短时间片 RR，后台长时间片 RR
- 调整每个人物的优先级：**模仿 SRTF**
  - 一开始最高优先级
  - 如果用完时间片还没完成，降低优先级
  - 如果时间片还没用完就完成了，提高优先级
- 算法的结果和 SRTF 相似：
  - 用时长的 CPU bound 任务会被很快降到低优先级
  - 用时短的 I/O bound 任务会留在高优先级队列
- 应用对算法的反制措施：插入短的无意义 I/O 以保持高优先级

许多调度器的 assumption：

- 经常睡眠，短 bursts => interactive 应用 => 高优先级
- 计算密集 => 低优先级

### Multi-Core Scheduling

**Affinity scheduling**：OS 最好总把线程调度到某个固定的 CPU 上运行（cache reuse）。

Spinlock:

```c
int val = 0;

Acquire() {
    while (test&set(val)) {}  // spin while busy
}

Release() {
    val = 0;    // atomic store
}
```

优点：

- **不需要上下文切换**，如果锁持有时间很短，性能比互斥锁好（睡眠并唤醒的开销过大）
- 适用于多个线程在 barrier 处等待的情况

但是每次 `test&set` 都是一次写，一个核的 `test&set` 会导致所有其他核的 cache 失效，导致 ping-pong。我们更希望 `test&test&set`：

```c
Acquire() {
    do {
        while (val) {}  // wait until might be free
    } while (test&set(val) == 1);  // exit if acquired lock
}
```

**Gang Scheduling**：多个线程完成同一个任务，将它们一起调度。

- 使得 spin-waiting 更有效率

Alternative: OS 通知并行应用其线程被调度到了多少个核上

- 应用适应其分配到的核数
- 核数增加对性能的提升是 sublinear 的，多应用 **space sharing** 更好

### Real-time Scheduling

**Real-time scheduling**（实时调度）：

- 目标：性能的**可预测性**
  - 实时系统中，性能是任务/类别中心的，被**先验**地保证
  - 常规系统中，性能是面向系统/吞吐量，是事后统计出来的
  - 实时系统需要可信地保证系统的最坏反应时间
- 硬实时：时间关键的安全导向系统
  - 必须满足所有 deadline（若可能）
  - 理想情况下，提前确定可行性（admission control）
  - **Earliest Deadline First (EDF)**
- 软实时：多媒体
  - 以高概率满足 deadline
  - Constant Bandwidth Server (CBS)

**Earliest Deadline First (EDF)**：

- 周期性任务 $i$，周期 $P_i$，执行时间 $C_i$，deadline $D_i^{t+1} = D_i^t + P_i$
- 每次都选择绝对 deadline 最急迫的任务执行
  - 抢占式调度：如果新任务的 deadline 比当前任务的 deadline 更早，则抢占当前任务
- **EDF feasibility testing**
  - 调度可行的充分条件：$sum_{i=1}^n \frac{C_i}{P_i} \leq 1$

### Ensuring Progress

Starvation：线程在一段不定时间内没有进展。

我们考察哪些调度算法会导致 starvation：

- Non-work-conserving 调度器
  - 即使有任务在 ready queue 中，调度器也可能让 CPU 空闲。这是导致 starvation 的调度算法的一个平凡解
- **非抢占式调度**都有 starvation 的问题
  - **Last-Come, First-Served**
    - 如果 arrival rate 大于 service rate，早来的任务就会饿死
  - **First-Come, First-Served**
    - 如果当前任务一直不 yield（如死循环），其他任务会饿死
- **Round Robin**：
  - 每个任务都确定地在至多 $(n-1)q$ 的时间内被调度
  - 无 starvation 问题
  - 从等待时间角度，RR 调度是公平的
- **优先级调度**容易导致低优先级任务的 starvation
  - 不过比这更重要的是优先级反转，它使得高优先级的任务也有可能饿死
    - 低优先级任务持有锁，高优先级任务请求此锁被阻塞，而中优先级任务阻塞低优先级任务释放锁
    - 低优先级任务持有锁，高优先级任务 `while (try_acquire(lock)) {}`
  - 解决方法：优先级捐献
    - 高优先级任务将其优先级捐献给它依赖的低优先级任务
  - SRTF 也是一种优先级调度，长任务可能会被饿死
  - MLFQ 是 SRTF 的近似，自然也有相同问题

### Case Study

**Linux $O(1)$ scheduler**：

- **Nice**: -20 ~ 19，nice 越小，优先级越高
- **优先级**：140 个优先级，值越小越优先
  - 0 ~ 99 是内核/实时任务
  - 100 ~ 139 是用户任务（priority = nice + 120）
- **所有算法都是 $O(1)$ 的**
  - 有一个 140 位的 bitmap 表示每个优先级是否有任务
- 两个优先级队列：active queue 和 expired queue
- active queue 中的任务用完时间片后会被放入 expired queue，所有任务都 expire 后两个队列交换
- 不同优先级的时间片大小也不同
- Heuristics:
  - 用户任务如果睡眠时间相比运行时间很长，说明它是 I/O bound 的，提升优先级
  - Interactive Credit: 睡眠很长则得到，运行很长则失去。
    - 作为一种滞后机制，防止突增突减触发不必要的切换
- 实时任务：
  - 总是抢占非实时任务
  - 优先级不会动态变化

**Proportional-Share Scheduling**：每个任务按优先级分配 CPU 份额（Lottery Scheduling）

- Lottery 调度的简单版机制：
  - 每个任务分得 $N_i$ 个彩票
  - 选取一个彩票编号 $d\in {1, \ldots, \sum_i N_i}$
  - 将 $N_i$ 排序，第一个满足 $\sum_i^j N_i > d$ 的 $j$ 号任务被调度

**Linux Completely Fair Scheduler (CFS)**：

- 基本思想：追踪每个线程的 CPU 时间，调度 CPU 时间少的线程以使它们追上平均 CPU 时间
- 任何时刻总选择 CPU 时间最少的任务执行，直到其不再是 CPU 时间最少的任务
- 使用一个 heap-like scheduling queue
  - 插入和删除 $O(\log n)$
- 睡眠中的线程，CPU 时间不会增长，因此它们被唤醒时会自动地被 boost
  - 自动实现了 interactivatity
- 目标：**Responsiveness/Starvation freedom**
  - 短的等待时间，并确保每个进程都得到进展
  - 约束：**Target Latency**（所有进程都得到服务的时间）
  - **时间片长度 = Target Latency / number of tasks**
- 目标：**Throughput**
  - 避免过多开销
  - 约束：**Minimum Granularity**（最短时间片长度）
- **Proportional shares**
  - Basic equal share：$Q_i = Target Latency \cdot 1/N$
  - Weighted share: $Q_i = Target Latency (w_i / \sum_j w_j)$
  - 利用 nice 值：$w = 1024 / (1.25)^{nice}$

### Choosing the Right Scheduler

| I Care About | Then Choose: |
| :----------: | :----------: |
| CPU throughput | FCFS |
| Avg. Completion Time | SRTF Approximation |
| I/O Throughput | SRTF Approximation |
| Fairness (CPU Time) | Linux CFS |
| Fairness (Wait Time to Get CPU) | Round Robin |
| Meeting Deadlines | EDF |
| Favoring Important Tasks | Priority |

解释：

- 吞吐量：FCFS 没有上下文切换开销，吞吐量最大
- 平均完成时间：SRTF 是平均完成时间最优的调度算法
- I/O 吞吐量：I/O 任务通常剩余时间很短，会被 SRTF 算法优先调度
- CPU 时间公平性：Linux CFS 使每个进程拥有大致相同的 CPU 时间
- 等待时间公平性：RR 确保每个进程都在 $(n-1)q$ 的时间内被调度

### Deadlock

死锁：对资源的循环等待。

两个 non-deterministic deadlock 的例子：

```c
// Thread A
x.acquire();
// 在这里切换到 Thread B
y.acquire();
y.release();
x.release();

// Thread B
y.acquire();
// 在这里死锁！
x.acquire();
x.release();
y.release();

// 由于内存空间有限的死锁
// Thread A
allocate_or_wait(1 MB);
// 在这里切换到 Thread B
allocate_or_wait(1 MB);
free(1 MB);
free(1 MB);

// Thread B
allocate_or_wait(1 MB);
// 在这里死锁！
allocate_or_wait(1 MB);
free(1 MB);
free(1 MB);
```

Dining Lawyers 问题：

- 五根筷子，五个律师
- 每个律师需要两根筷子才能吃饭
- 如果每个律师同时抓住一根筷子，则没有律师可以吃饭 => 死锁！
- 解决方法：
  - 让某个律师放弃一根筷子，则另一个律师可以开始吃饭
  - 等他吃完后，就不会再有死锁了
- 避免死锁：
  - 如果拿走最后一根筷子会导致此后没有人能够持有两根筷子吃饭，则不允许拿走最后一根筷子

发生死锁的四个条件：

1. Mutual exclusion：资源只能被一个线程同时使用
2. Hold and wait：持有资源的线程正在等待获取其他被其他线程持有的资源
3. No preemption：资源只能被持有的线程在用完后主动释放
4. Circular wait：存在一个线程的循环等待链 $\{T_1, T_2, \ldots, T_n\}$，其中 $T_i$ 等待 $T_{i+1}$ 持有的资源，$T_n$ 等待 $T_1$ 持有的资源

Resource-Allocation Graph：

- 系统模型：
  - 线程 $T_1, T_2, \ldots, T_n$
  - 资源种类 $R_1, R_2, \ldots, R_m$
  - 每种资源有 $W_i$ 个实例
  - 每个线程以 `request()`、`use()`、`release()` 的方式使用资源
- Resource-Allocation Graph：
  - 结点：$T_i$ 和 $R_j$
  - 请求边：$T_i \to R_j$，表示 $T_i$ 请求 $R_j$ 的资源
  - 分配边：$R_j \to T_i$，表示 $R_j$ 被 $T_i$ 持有
- 图中有死锁则一定有环，但是有环不一定有死锁
- 死锁检测算法：
  - 思路：可以轻易地得知一个线程是否能就绪
    - 只要它请求的资源都空闲
  - 从图中删除所有这样的就绪线程，如果还剩下线程，则说明存在死锁

```c
// Deadlock Detection Algorithm
Array<int> avail;    // Free resource counts for each resource type
Set<Thread> threads = all_threads;

do {
    done = true;
    for (Thread t : threads) {
        // t.request is an array of m,
        // representing number of each resource type t needs
        if (t.request <= avail) {  
            // Thread t can finish
            avail += t.request;    // Release resources held by t
            threads.remove(t);
            done = false;
        }
    } 
} while (!done);

bool is_deadlocked = (threads.size() > 0);
```

系统解决死锁的方法：

1. Deadlock prevention：
   - 一开始就不写出来会死锁的代码
2. Deadlock recovery
   - 让死锁发生，并设法从中恢复
3. Deadlock avoidance
   - 动态地推迟资源请求，从而避免死锁发生
4. Deadlock denial
   - 掩耳盗铃，忽略死锁
   - 反正出问题了重启就完了

现代操作系统确保系统中没有死锁（deadlock prevention），忽略应用程序中的死锁（deadlock denial）。

预防死锁的方法：

- 无限资源
  - 虚拟内存
- 不允许共享资源
- 不允许 wait
  - 电话公司
  - 计算机网络（if collision, back off and retry）
- 令所有线程一次性请求所有需要的资源
  - 原子的 `acquire_both(x, y)`
- 强迫所有线程都按某个特定顺序请求资源
  - 释放的顺序无所谓

从死锁中恢复的方法：

- 终止线程，强迫其放弃资源
- 抢占资源
  - 虚拟内存的机制也可以视为抢占内存资源
  - 操作系统将暂时不用的内存 page out 到磁盘，就是抢占了这块内存资源
- 回滚死锁了的线程

避免死锁的方法：

- Naive 方法：当线程请求资源时，OS 检查这次请求是否会导致死锁
  - 一次请求可能不会直接导致死锁，但可能导致未来无可避免地陷入死锁
- 三种状态
  - Safe 状态：系统可以推迟资源获取以预防死锁
    - **系统存在一个线程执行顺序，使得此顺序下不会发生死锁**
  - Unsafe 状态：尚未死锁，但线程的请求可能会导致未来无可避免地陷入死锁
  - Deadlocked 状态：系统已经死锁了（deadlcoked state 也是 unsafe state）
- 理念：当线程请求资源时，OS 检查这次请求是否会导致系统进入 unsafe 状态
  - 如果会，则使线程等待其他线程释放资源

```c
// 之前的例子
// Thread A
x.acquire();
// 在这里切换到 Thread B
y.acquire();
y.release();
x.release();

// Thread B
// 这次资源获取会导致 unsafe 状态！
// 在此处等待
y.acquire();  
x.acquire();
x.release();
y.release();
```

银行家算法：

- 线程事先声明它的最大资源需求量
- 当线程请求资源时，银行家算法先假设此次请求被批准，然后运行死锁检测算法，若不会发生死锁，则批准
- 系统会一直处于 safe 状态

```c
Map<Thread, Array<int>> max;    // Maximum resource counts for each thread
Map<Thread, Array<int>> alloc;    // Allocated resource counts for each thread
Array<int> avail;    // Free resource counts for each resource type

if (t.request > (max[t] - alloc[t]) || t.request > avail) 
    return false;


// If thread t requests resources
alloc_sim = alloc.copy();
alloc_sim[t] += t.request; 
avail_sim = avail - t.request;
Map<Thread, Array<int>> need_sim = max - alloc_sim;

// Check if the system is in a safe state
return !detect_deadlock(need, avail_sim);
```

对律师就餐问题，银行家算法给出的解决方案：

- 如果拿的不是最后一根筷子，则批准拿走
- 如果拿的是最后一根筷子，但拿走后仍然有其他人能吃饭，则批准拿走
- 假如律师有 $k$ 只手
  - 如果拿的是最后一根筷子，且拿走后没有人能拿够 $k$ 根筷子，则不批准拿走
  - 如果拿的是倒数第二根筷子，且拿走后没有人能拿够 $k-1$ 根筷子，则不批准拿走
  - ...

### ZygOS

ZygOS: Achieving Low Tail Latency for Microsecond-scale Networked Tasks

场景：serve us-scale RPCs

- 应用：KV-stores、In-memory DB
- 数据中心环境：fan-out/fan-in（一个人给很多人发消息/一个人收到很多人的消息）
- Tail-at-scale 问题：
  - 一个请求分成若干子请求，大多数子请求延迟很低，少数请求的延迟很高，拉高了总延迟（总延迟 = max(子请求延迟)）
- 目标：在一个激进的尾延迟 service-level objectives (SLO) 下提高吞吐量
- 方法：对于叶结点
  - 减少系统开销
  - 调度

Queueing theory:

- Processor
  - FCFS
  - Processor sharing (RR)
- 多队列/单队列
  - 每个核一个队列还是所有核共享一个队列
- Inter-arrival 分布：泊松分布
- Service time 分布：
  - Fixed
  - 指数
  - Bimodal
- 无系统开销，服务时间独立，性能上界

Baseline：

- Linux：
  - Partitioned connection delegation
    - 每个核一个队列  
    - 非 work-conserving：某个核空闲时不会从其他核的队列中取任务
  - Floating connection delegation
    - 每个核一个队列  
    - Work-conserving
- Dataplanes:
  - 与 Linux (partitioned conn) 不同，许多工作在用户空间完成，没有内核-用户上下文切换开销
- ZygOS 的目标：Dataplanes + Linux (floating conn)

执行模型：

- Shuffle layer
  - 每个核有自己的 shuffle queue，当队列空时，可以从其他核的 shuffle queue 中偷取任务
  - 偷取完的任务通过 shuffle layer 归还任务原主人，由原主人还给网络层
- 通过 shuffle layer 使得多队列表现收敛到单队列表现

### Tiresias

挑战：

- 调度：不可预测的训练时间
- 任务放置：过于激进的 job consolidation 会造成 GPU 碎片化和较长的 queueing delay

方法：

- Discretized 2D Age-Based Scheduler
  - 每次调度选择 GPU 时间最少的任务执行
  - GPU 时间 = 执行时间 * 占用 GPU 核数
  - 以时间片为单位，避免频繁上下文切换
  - 本质上 MLFQ 变种
- Model profile-based placement
  - 如果模型的 tensor size 是 highly skewed 的，则需要 consolidation

实验结果媲美 SRTF。

### DRF

Fair-sharing：

- 每个用户获得 $1/n$ 的资源
- 泛化：max-min fairness
  - 每个用户获得 $1/n$ 的资源，除非它不需要这么多
- 再泛化：weighted max-min fairness
  - 每个用户获得 $w_i / \sum_j w_j$ 的资源，除非它不需要这么多

Fairness 的定义：

- Share guarantee
  - 每个用户至少获得 $1/n$ 的资源，除非它不需要这么多
- Strategy-proof
  - 用户没有动机谎报更多需求
- Pareto efficiency

问题：如果**公平地**为**不同的需求**分配**多种资源**？

模型：需求向量 $<2, 3, 1>$

Natural policy:

- Asset fairness: 每个用户所有种类的资源的简单加和是相等的
  - 不满足 share guarantee
- Dominant resource fairness
  - dominant resource：使得 $\frac{资源占有量}{资源总量}$ 最大的资源种类
  - dominant share：用户占优资源的 $\frac{资源占有量}{资源总量}$
- 在 dominant share 上应用 max-min fairness
  - 不同用户的 dominant share 相等，除非它们不需要这么多
  - 可以证明此策略满足 share guarantee、strategy-proof 和 Pareto efficiency

Competitive Equilibrium from Equal Incomes (CEEI)：

- 每个用户相同的初始禀赋
- 他们会通过交易达到均衡
- 但这不是 strategy-proof 的（不如 DRF 公平）

### FairRide

模型：

- 用户按照固定速率访问相等大小的文件
  - $r_{ij}$: 用户 $i$ 访问文件 $j$ 的速率
- Allocation policy 决定选择哪些文件放入缓存
  - $p_j$：文件 $j$ 被缓存的比重
- 用户关心其缓存命中率 $HR_i = \frac{total\_hits}{total\_accesses} = \frac{\sum_j p_j r_{ij}}{\sum_j r_{ij}}$

性质：

- Isolation guarantee (share guarantee)
  - 没有用户的状况比 static allocation 更差
- Strategy-proofness
  - 用户没有动机谎报访问速率
- Pareto efficiency

定理：没有分配策略能同时满足这三个性质

FairRide:

- 满足 isolation guarantee 和 strategy-proofness
- 达到近似最优的 Pareto efficiency
- 方法：
  - 为每个用户分配 $1/n$ 的禀赋
  - 多个用户分享同一个文件，则它们平分访问的 cost 
  - 阻塞不付费的用户访问（实现为 delaying）
  - 以 $p(n_j) = \frac{1}{n_j} + 1$ 的概率阻塞用户访问，其中 $n_j$ 是缓存文件 $j$ 的用户数
    - 例如 $p(1) = 0.5$
- 作弊总会得到更坏的结果