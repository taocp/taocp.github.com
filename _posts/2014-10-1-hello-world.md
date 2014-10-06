---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- linux
- programming
- OS
categories:
- "linux"
---

    #include <stdio.h>
    int main(void)
    {
        printf("hello, world\n");
        return 0;
    }

"hello, world"是我编程生涯中的第一个程序，
读了3年大学，回过头来看看"hello, world"的背后究竟发生了什么。

限于能力和时间，没有针对每一个细节展开地很详细。
要不然可以顺着把整个操作系统、编译原理揪出来。

需要一定的C语言、操作系统、编译原理、计算机网络、Linux/shell基础。

**Linux v0.11 kernel为例**

===
# taocipian@x1-carbon ~ %
现在shell已经准备就绪，打印出一个"$"或者"%"或者其它符号作为提示符，
接着调用`fgets()`(或者其它类似函数)来读取用户输入的命令，
`fgets()`由标准库实现，可能还要调用若干其它函数，不过最终会到达`read()`。

这个`read()`(`man 2 read`)也是由标准库提供的，一般我们称它为"system call"，
但其实这个`read()`不是真正的"system call"，看看一个简单的`read()`实现。

    int read(int fd, void *buf, int count)
    {
        int res = 0;
        asm(
            "movl  $3, %%eax\n\t"
            "movl  %1, %%ebx\n\t"
            "movl  %2, %%ecx\n\t"
            "movl  %3, %%edx\n\t"
            "int $0x80\n\t"
            "movl  %%eax, %0\n\t"
            :"=m"(res)
            :"m"(fd), "m"(buf), "m"(count)
            );
        return res;
    }

可见这个`read()`是通过`int 0x80`陷入内核，调用内核里的`sys_read()`来交差的，后者才是真正的"system call"。

内核中，`sys_read()`会针对具体的对象调用不同的、具体的read函数，
例如read文件，则`sys_read()`会接着调用`file_read()`；
如果read块设备，则`sys_read()`会接着调用`block_read()`。

此时shell是等待键盘输入，于是`read()`会接着调用`rw_char()`，
而`rw_char()`经过层层调用，最终会跳到`tty_read()`，
`tty_read()`会把tty设备输入队列中的数据复制到用户缓冲区buf中，
也就是`int read(int fd, void *buf, int count)`中的那个`buf`。

    // tty_read() 的代码片段：
    if (EMPTY(tty->secondary) ||
            (L_CANON(tty) && !tty->secondary.data && LEFT(tty->secondary)>20)) {
        sleep_if_empty(&tty->secondary);
        continue;
    }

此时输入缓冲队列(名为secondary)为空，没有数据来源可以复制，
于是shell进程主动sleep让出CPU，内核会在某个恰当时机唤醒shell进程。

  - 背景知识：tty设备的输入队列

为了方便叙述，简化背景，定义：tty设备=键盘+显示器。

tty设备有2个队列用于存储输入，分别是`read_q`和`secondary`。

`read_q`对输入不做任何处理，输入什么存储什么；

`secondary`则对输入做一些必要的处理，**cooked mode**，行缓冲。

例如，
假设用户输入了"ABC<Backspace\>D"，
则`read_q`中为"ABC<Backspace\>D"，`secondary`中则为"ABD"。

（行）缓冲的输入设备是最常见的情形，
[怀着对使用非缓冲输入的朋友的歉意，我们假设您在使用缓冲输入](http://book.douban.com/subject/1240002/)。

回到大标题"taocipian@x1-carbon ~ %"，

一句话解释：现在shell因为等待I/O而休眠了。

===
# % vi helloworld.c
  - 按键回显

用户按下一个键时，就会产生键盘中断请求信号，发送给中断请求控制器。
CPU在一个指令周期的最后阶段会检测中断请求，假设此时键盘中断优先级最高，CPU给予响应。

控制跳转到键盘中断处理程序`keyboard_interrupt`，
它做一些处理(例如将按下的键加入到读缓冲队列`read_q`中)，之后会调用`do_tty_interrupt()`，
`do_tty_interrupt()`没有其它任何操作，直接跳到`copy_to_cooked()`，
将读缓冲队列`read_q`中的数据按照[**cooked mode**](https://en.wikipedia.org/wiki/Cooked_mode)
进行处理后放进辅助队列`secondary`里，
在处理过程中，如果设置了回显（默认），则将该字符加入到写缓冲队列`write_q`中，
**再调用`con_write()`将该字符显示在屏幕上。**
处理完后，唤醒所有等待`secondary`队列的进程，其中就包括了shell。

`copy_to_cooked()`遇到一个换行符或者EOF就会增加变量`tty->secondary.data`的值，以此记录输入缓冲中有几个输入行（行缓冲机制以行为单位交付给上层应用程序）。


  - shell获取键入内容

承前所述，shell已经被唤醒处于就绪态，但暂时没有被分配到CPU。在某一次调度中，CPU选择shell继续执行。

    // tty_read() 的代码片段：
    if (EMPTY(tty->secondary) ||
            (L_CANON(tty) && !tty->secondary.data && LEFT(tty->secondary)>20)) {
        sleep_if_empty(&tty->secondary);
        continue;
    }

`tty_read()`从`sleep_if_empty()`中返回后`continue`到循环首部再次执行到`if()`语句处检查条件，
发现缓冲队列`secondary`中还不够一行字符(`tty->secondary.data == 0`)，于是再次`continue`。
用户接着输入，直到按下了一个回车，条件成熟，于是从`secondary`队列中将键入的内容复制到用户缓冲区`buf`中。`tty_read()`在读取到需要的字节数或者行缓冲模式下读取到换行符就会返回。

这样，当用户输入`vi helloworld.c`并按下回车键后，`tty_read()`返回到内核中的`sys_read()`，接着中断返回到标准库的`read()`，一路向上直到`fgets()`返回，继续执行用户代码，此时输入的`vi helloworld.c`已经被shell获取。

  - 显示字符的原理
    - console
    - X(X的原理)
    - wayland

  - shell解释、执行

    - 对于用户输入的命令行（此处是`vi helloworld.c`），shell按一定流程进行处理，最终`fork()`出子进程，
再调用`execve()`执行`vi`，自己则等待`vi`的结束。
    - shell命令行处理步骤：[《shell脚本学习指南》](http://book.douban.com/subject/3519360)P179解释很详细。

  - file I/O

`vi`获取用户输入的字符，写入文件`helloworld.c`中。

===
# % gcc -Wall helloword.c
  - 编译：预处理、编译、汇编、链接

// TODO 短时间内不会写，编译原理还需下功夫。

===
# % ./a.out
  - why `./`

一句话：安全考虑。

直接引用[PATH-wiki](https://en.wikipedia.org/wiki/PATH_(variable\))中的一段：

The current directory (`.`) is sometimes included by users as well, allowing programs residing in the current working directory to be executed directly. Superuser (root) accounts as a rule do not include it in `$PATH`, however, in order to prevent the accidental execution of scripts residing in the current directory, such as may be placed there by a malicious tarbomb.

  - 装载

用户输入`./a.out`并回车后，shell会先`fork()`出子进程，子进程再`execve()`执行`a.out`。
`fork()`、`execve()`等函数和`read()`类似的道理，由标准库往下陷入内核。

`execve()`在内核中对应`sys_execve()`，它会读取可执行文件`a.out`的前128个字节，用以判断可执行文件的格式。
如果可执行文件的第一行内容诸如`#!/bin/bash`则交由bash处理。

<!--// TODO 动态、静态-->

此处`a.out`是elf格式的二进制可执行文件（并且还是动态链接，gcc默认），
所以`sys_execve()`最终会调用到`load_elf_binary()`。

    // 因为 Linux v0.11 还不支持动态链接
    // 所以参考 Linux v3.16.0 的部分代码
    // fs/binfmt_elf.c
    static int load_elf_binary(struct linux_binprm *bprm)
    {
        // ...
        if (elf_interpreter) {
            // ...

            elf_entry = load_elf_interp(/*...*/);

            // ...
        }
        // ...
    }

于是内核根据`a.out`的`.interp`段找到动态链接器，把它映射到进程的虚拟地址空间。
并且把`sys_execve()`的返回地址修改为动态链接器的入口地址。
当`sys_execve()`中断返回时控制就跳转到了动态链接器，
接着动态链接器自举，然后载入`a.out`依赖的各个共享库，
完成后跳转到`a.out`本身的入口开始执行。

  - 运行

`a.out`真正的入口不是`main()`，可能是`_start`，这取决于runtime的实现。

    #include <stdio.h>
    int main(void)
    {
        printf("hello, world\n");
        return 0;
    }

`_start`做一些初始化工作，例如准备`main()`所需的参数`int argc, char *argv[]`，初始化堆等等。
然后调用`main()`，接着调用`printf()`，一顿狂奔到达`write(/*stdout*/)`陷入内核`sys_write()`，
最终跳到`tty_write()`调用`con_write()`通过汇编语句将"hello,world"写在屏幕上。

然后逐级向上返回直到`main()`，`return 0`后离开`main()`再次来到`_start`，
做一些可能的清理工作，然后`exit()`再次陷入内核`sys_exit()`，
释放占用的内核资源（如页表），将自己的进程状态设置为ZOMBIE，
通知父进程，然后`schedule()`调度执行别的进程，这次`schedule()`离开后就永远也不会再回来执行了。


    int sys_waitpid(pid_t pid,unsigned long * stat_addr, int options)
    {
        // ...
        case TASK_ZOMBIE:
            //...
            release(*p);
            //...
        // ...
    }

    void release(struct task_struct * p)
    {
        int i;

        if (!p)
            return;
        for (i=1 ; i<NR_TASKS ; i++)
            if (task[i]==p) {
                task[i]=NULL;
                free_page((long)p);
                schedule();
                return;
            }
        panic("trying to release non-existent task");
    }

父进程shell`wait()`子进程再做最后的清理，例如释放 子进程PCB及内核栈 使用的1页内存。

至此，"hello, world"彻底完结。

shell等待`a.out`终止以后又打印出一个%提示符，等待用户的下一次输入。

===

参考资料

[Linux v0.11](https://www.kernel.org/pub/linux/kernel/Historic/old-versions/)

[《shell脚本学习指南》](http://book.douban.com/subject/3519360/)

[《Linux内核完全剖析》](http://book.douban.com/subject/1481173/)

[《程序员的自我修养》](http://book.douban.com/subject/3652388/)

[《链接器和加载器》](http://book.douban.com/subject/4083265/)

[《C Primer Plus》](http://book.douban.com/subject/1240002/)
