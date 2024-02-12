
# Getting to know asyncio

## I/O-bound和CPU-bound

主要看系统的bottleneck在哪里，在I/O就是I/O-bound，在CPU就是CPU-bound。

## 进程（process）和线程（thread）
进程是资源分配的最小单位，线程是CPU调度的最小单位。

## Concurrency、Parallelism和Multitasking

两个单词本身都有同时做多件事的语义，但Concurrency可以包括时分复用，而Parallelism是真正意义上的并行做多件事，所以估计这只是一种约定的叫法。

除了这俩名字，还有一个叫multitasking的，是指一个系统能够同时做多件事。感觉这个概念是从操作系统的角度来描述的，即一个操作系统的单任务还是支持多任务。最最简单的操作系统可能只支持单任务，比如先进先出的简单任务调度策略。但现在的操作系统基本都是多任务系统，在单核的CPU上，只能通过时分复用来实现多任务，这就对应了Concurrency；而多核的CPU上，就可以通过真正的并行来支持多任务，这就对应了Parallelism。

有两种multitasking的策略，即preemptive multitasking和cooperative multitasking。前者由操作系统主导，决定下一个要执行的任务及其时间；后者由程序员自己决定，即程序员自己决定下一个要执行的任务及其时间。由于操作系统是不感知程序的语义的，所以它的策略对程序来说是随机的，而程序员自己决定策略，就相当于程序员自己控制了程序的语义，所以程序员自己决定策略的程序，就叫做cooperative multitasking。

现在一般把cooperative multitasking中调度的单位叫做coroutine。coroutine和process/thread都需要做context switching，但coroutine的context switching是程序员自己控制的，而process/thread的context switching是操作系统控制的，对应前面提到的cooperative multitasking和preemptive multitasking。

## 线程（thread）和协程（coroutine）的Context Switching

相比线程，协程的context switching更轻量级，这主要是由实现方式决定的。

线程的context switching是操作系统控制的，由操作系统来保存和恢复线程的上下文，涉及操作系统kernel模式下的操作，涉及到的上下文则包括了程序运行时的各种状态，比如CPU寄存器、栈指针、堆栈、程序计数器（Program Counter有时也称为Instruction Pointer）等，所以context switching的开销很大。

而协程的context switching是程序员自己控制的，由程序员自己来保存和恢复协程的上下文，完全在用户态下进行，所以context switching的开销很小。

下面是ChatGPT给出的详细解释：

> ### Traditional OS-Managed Context Switching (Threads/Processes):
> In the context of threads or processes, when the OS performs a context switch:
> 
> * **All Registers are Saved/Restored:** This includes the program counter (PC). The OS saves the current state of all CPU registers, including the PC, of the currently running thread/process into its context structure (like a Process Control Block for processes). Then, it restores the state of the CPU registers, including the PC, for the next thread/process to run. This switch changes which piece of code the CPU executes next.
> 
> ### Coroutine Switching in User Space:
> 
> When it comes to coroutines, particularly stackless coroutines managed within a user-level library (like Python's `asyncio`):
> 
> * **"Program Counter" at User-Level:** The concept of the "program counter" gets abstracted at the user level. Coroutines are functions with the ability to pause (yield) and resume. The point at which a coroutine yields (awaits) can be thought of as its "program counter" in a metaphorical sense because it marks the spot where execution will resume.
> 
> * **Not a CPU Register Operation:** Unlike a thread or process switch, switching between coroutines does not involve saving and restoring the actual CPU program counter. Instead, it involves saving and restoring coroutine-specific data that allows the coroutine to resume where it left off. This includes the coroutine's local variables and its execution state (e.g., what it is waiting on).
> 
> * **Controlled by User-Level Code:** The switching mechanism is orchestrated by the coroutine library/runtime, not the OS. For instance, when a coroutine yields control because it is waiting for I/O, the coroutine scheduler (part of the user-level library) saves its state and decides which coroutine to execute next. This switch is much more lightweight than an OS-level context switch because it doesn't involve kernel transitions or saving/restoring the entire CPU state.


我的感觉是，协程的实现一定程度上依赖了编程语言将函数（function）作为一等公民（first-class citizen）的能力，这样就能在用户态实现函数的调度。

## 关于global interpreter lock（GIL）

在CPython实现上，每一个Process都有一个GIL，GIL的本质是一种锁，防止多个线程同时修改object的reference count，因此GIL的存在限制了同一时间只能有一个线程在执行，即使运行在多核CPU上。

但存在一种特殊的例外，就是I/O操作，因为I/O操作会导致GIL释放，也就是说在某个线程执行I/O操作时，GIL会释放，从而让其他线程执行。

## Socket

所有I/O操作都是通过Socket来实现的，比如读取文件，发送HTTP请求等。在操作系统层面，都有一种机制来实现异步I/O，比如Linux的epoll，Windows的IOCP。

参考ChatGPT

> ### How epoll Works:
> Registration of File Descriptors: With epoll, you register file descriptors (like sockets) with the epoll instance.
> 
> Monitoring for Events: The epoll instance monitors these file descriptors for specified events, such as data available for reading (EPOLLIN), the ability to write without blocking (EPOLLOUT), or an error condition (EPOLLERR).
> 
> Wait for Events: The program can then wait for events to occur on any of the registered file descriptors. This is typically done using the epoll_wait system call, which allows the program to efficiently wait for multiple events simultaneously.
> 
> Handling Events: When events occur on the registered file descriptors, the program can react accordingly, such as reading or writing data, accepting connections, or handling errors.

借助这样的机制，Python就可以实现异步I/O了。