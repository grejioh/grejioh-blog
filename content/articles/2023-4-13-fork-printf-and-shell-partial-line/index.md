---
title: "从 fork() 和 printf() 开始，看 shell 是如何处理部分行"
date: "2023-04-13T22:00:00+08:00"
toc: true
tags: ["shell","C"]
katex: true
categories: ["Tech"]
---

观察这么一段代码：  

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    int i;
    for(i=0; i<2; i++) {
        fork();
        printf("-");
    }
    return 0;
}
```
编译运行改代码，输出应该是什么?

在实际运行时，这些都是我得到的结果：

```
--------
------
--
----
```
然而从理论上讲，答案是：--------，八个-。

尝试一下，为什么是这样？或者为什么不是这样？

这里给出结论，这段代码具体的输出以下因素有关：机器的性能（核心数、线程数）、用户选用的 shell。改变一些代码可以得到比较稳定的输出。下面给出分析。  

## 理论分析
在分析前，我们需要对 fork() 有一点了解，下面是 BingAI 的表述：  
>在使用fork()系统调用创建子进程时，会复制父进程的整个地址空间并把复制的那一份分配给子进程。这包括进程上下文、进程堆栈、内存信息、打开的文件描述符、信号控制设置、进程优先级、进程组号、当前工作目录、根目录、资源限制、控制终端等。但是，由于写时复制技术的引入，只有在进程空间的各段的内容要发生变化时，才会将父进程的内容复制一份给子进程。  

我们假设在一个单线程的机器上进行测试，则存在这样一种调度方式：

1. 用户在终端使用 shell 创建了一个 pid 为1230的进程，我们称 p0
2. p0 循环 i==0
    1. p0 通过 fork()创建了 pid = 1231 进程，我们称p1，p1 处于就绪状态（runnable state）
    2. p0 进程调用 printf("-")
3. p0 循环 i==1
    1. p0 通过 fork()创建了 pid = 1232 进程，我们称p2，p2 处于就绪状态（runnable state）
    2. p0 进程调用 printf("-")
4. p0 进程 return 0
5. p1 进程从 fork() 后开始执行，调用 printf("-")
6. p1 进程循环 i==1
    1. p1 通过 fork()创建了 pid = 1233 进程，我们称p3，p3 处于就绪状态（runnable state）
    2. p1 进程调用 printf("-")
7. p1 进程 return 0
8. p2 进程从 fork() 后开始执行，调用 printf("-")
9. p2 进程return 0
10. p3 进程从 fork() 后开始执行，调用 printf("-")
11. p3 进程 return 0
> shell 创建 p0
>
> p0 创建 p1 与 p2
>
> p1 创建 p3

观察上述过程，我们发现共调用printf("-")6次。

那么为什么，理论结果是 8 次而不是 6 次？  

### printf() 与流缓冲机制
printf() 函数将指定的字符串和变量格式化后输出到标准输出流（stdout）。

在 Stream Buffering (The GNU C Library) 中讲到：  

>写入流的字符通常会积累并以块的形式异步传输到文件，而不是在应用程序输出时立即出现。  

并且还提到在以下情况下流会刷新：  
1. 当尝试进行输出并且输出缓冲区已满时。
2. 当流关闭时。
3. 当程序通过调用 exit 终止时。
4. 写入换行符时（如果流是行缓冲的）。
5. 每当对任何流的输入操作实际从其文件中读取数据时。
6. 以及，如果想刷新缓冲输出，可以调用 fflush。

具体的原理请参阅Flushing Buffers (The GNU C Library) 。  

注意这段代码printf()函数的参数并没有包含\n换行符，因此刷新流缓冲区时，也就是说我们的程序真正将-输出到屏幕上时，是在程序正常终止后（在main函数中执行 return 0也是正常终止）。  

printf("-")将"-"写入了父进程缓冲区，位于父进程的堆栈空间中，当我们使用fork()函数创建子进程时，子进程复制了一份父进程的堆栈空间，也把缓冲区中已经存在的"-"复制过去了。

对这段修改后的代码理论分析，可见一斑：  

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    int i;
    pid_t ppid = getpid();
    for(i=0; i<2; i++) {
        fork();
        printf("%d ", getpid()-ppid);
        //printf("-");
    }
    return 0;
}
```
ppid 保存的是 p0 的 pid，则printf("%d ", getpid()-ppid)就可以反映出是哪个进程写入的缓冲区。

按照之前的调度，该段代码的理论结果是：0 0 1 1 0 2 1 3。

根据调度策略的不同，别的顺序也合理，比如：0 0 0 2 1 1 1 3。

这些数字我们两两一组来看：  

- 0 0 是由 p0 输出
- 1 1 是由 p1 输出
- 0 2 是由 p2 输出，反映 p2 的缓冲区复制自 p0
- 1 3 是由 p3 输出，反映 p3 的缓冲区复制自 p3

由此，我们便确定了理论的答案：--------，八个-。

## 现实与理论的差别
然而我们将代码跑在不同的机器上，跑在不同的 shell 环境中，会有不同的结果，并且结果还不稳定。下面分别说明。

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    int i;
    for(i=0; i<2; i++) {
        fork();
        printf("-");
    }
    return 0;
}
```
### 下面是在四种不同情况下运行以上程序多次的结果：
#### 1 cpu 核心的性能孱弱的 Linux vps 上，shell 为 bash
```shell
bash-3.2 ./plain.out
--bash-3.2$ ------
bash-3.2 ./plain.out
--bash-3.2$ ------
bash-3.2 ./plain.out
--bash-3.2$ ------
bash-3.2 ./plain.out
--bash-3.2$ ------
```
bash 会在 p0 exit 后打印提示符（“bash-3.2$ ”），然后其他三个子进程 exit，打印剩余内容，由于为单核单线程机器，调度行为较为稳定，每次结果都一致。
#### 1 cpu 核心的性能孱弱的 Linux vps 上，shell 为 zsh
```shell
❯ ./plain.out
--#

❯ ./plain.out
--#

❯ ./plain.out
--#

❯ ./plain.out
--#
❯
```
由于 zsh 的默认选项，剩余六个 ------- 被提示符覆盖了，由于为单核单线程机器，调度行为较为稳定，每次结果都一致。zsh 默认为部分行（partial line，指不含有换行符的行）补充符号#或$，# 代表 root 用户，下面看到的 $ 代表非 root 用户。
#### 性能不错的本地机器，支持多线程并发执行，shell 为 bash
```shell
bash-3.2$ ./plain.out
----bash-3.2$ ----

bash-3.2$ ./plain.out
----bash-3.2$ ----

bash-3.2$ ./plain.out
--------bash-3.2$

bash-3.2$ ./plain.out
----bash-3.2$ ----

bash-3.2$ ./plain.out
----bash-3.2$ ----
```
由于是多核心机器，可以并发执行多个进程，在 p0 exit 后，bash打印提示符（“bash-3.2$ ”）之前，又有若干子进程 exit，完成打印。在大部分情况下，所有进程（p0～p3）都 exit 了，少部分情况下，有一到三个未完成的子进程，因此会将部分-输出在提示符后面。
#### 性能不错的本地机器，支持多线程并发执行，shell 为 zsh
```shell
❯ ./plain.out
--------%

❯ ./plain.out
------%

❯ ./plain.out
--------%

❯ ./plain.out
--------%

❯ ./plain.out
------%

❯ ./plain.out
--------%
```

由于是多核心机器，可以并发执行多个进程，在 p0 exit 后，bash打印提示符（“❯ ”）之前，又有若干子进程 exit，完成打印。在大部分情况下，所有进程（p0～p3）都 exit 了，少部分情况下，有一到三个未完成的子进程，因此会将部分-输出在提示符后面，乃至被提示符覆盖。
### 分析观察结果
#### 机器性能的影响
机器性能影响在进程 p0 exit 后，提示符打印前，有多少进程可以完成打印。

这产生了不稳定性，为了解决这个问题有以下方案：

##### 让进程 p0 exit 前 sleep 等待其他子进程执行完毕
```c
#include <stdio.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    int i;
    pid_t ppid = getpid();
    for(i=0; i<2; i++) {
        fork();
        printf("%d ", getpid()-ppid);
       	// printf("-");
    }
    if (ppid == getpid()) {
        sleep(1);
    }
    return(0);
}
```
##### 让每个进程 exit 前 wait，等待对应子进程执行完毕
```c
#include <stdio.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    int i;
    pid_t ppid = getpid();
    for(i=0; i<2; i++) {
        fork();
        printf("%d ", getpid()-ppid);
        //printf("-");
    }
    //if (ppid == getpid()) {
    pid_t wpid;
    int status;
	while ((wpid = wait(&status))>0);
    //}
    return(0);
}
```
注意，如果只让 p0 wait 也不够稳定，因为 p0 exit 时可能 p3 还未 exit。 
#### bash 和 zsh 的影响
shell 会在收到 p0 的返回值后输出提示符（“❯ ” 或“bash-3.2$ ”）。

bash 默认不会为没有换行符的部分行（partial ）补空格或做特殊处理，因此产生了这种输出：

```shell
bash-3.2$ ./plain.out
----bash-3.2$ ----
```

但 zsh 比较特殊，通过阅读文档，发现 zsh 默认设置了两个选项（详见：zsh: 16 Options）：

##### PROMPT_CR
>在行编辑器中打印提示符之前打印一个回车符。这是默认选项，因为只有当编辑器知道行的开头出现在哪里时，才可以进行多行编辑（multi-line editing）。

意思是，为实现 zsh 提供的多行编辑功能，zsh 需要知道每次用户输入的指令的开头位置。为了知道这个位置，zsh强制通过在打印提示符前打印回车符（注意是只回车，并不换行），将“光标”移到行的开头，然后打印提示符。这可能会覆盖部分行（partial line）。

##### PROMPT_SP
> 尝试保留部分行（partial line，即不以换行符结尾的行），否则由于  PROMPT_CR  选项，该行将被命令提示符覆盖。这是通过输出一些光标控制字符（包括一系列空格）来实现的，这些字符应该使终端在存在部分行时换行到下一行（请注意，这只有在您的终端具有自动边距时才会成功，这是典型的）。
> 
> 保留部分行时，默认情况下，您会在部分行的末尾看到一个反色+粗体字符：“  %  ”代表普通用户，“  #  ”代表 root。如果设置，shell 参数  PROMPT_EOL_MARK  可用于自定义部分行结尾的显示方式。
>
>注意：如果未设置  PROMPT_CR  选项，则启用此选项将无效。默认情况下此选项处于启用状态。

意思是：在设置了 PROMPT_CR 后，部分行会因为回车（不是换行），而被覆盖，为了解决这个问题，设置PROMPT_SP 可以在部分行后输出一些控制字符，保留这个部分行。
##### zsh 取消设置 PROMPT_CR（隐含了取消设置 PROMPT_SP）
```shell
❯ setopt NO_PROMPT_CR
❯ ./plain.out
--------❯ 
```
可以看到，提示符直接出现在--------后面。
##### zsh 设置 PROMPT_CR，但取消设置 PROMPT_SP
```shell
❯ setopt PROMPT_CR
❯ setopt NO_PROMPT_SP
❯ ./plain.out
❯ 
```
可以看到，所有的输出均被覆盖。

## 总结
最好的解决方案就是不要写这样的程序。  

