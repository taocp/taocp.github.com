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

**Linux v0.11 kernel为例**

"hello, world"是我编程生涯中的第一个程序，
读了3年大学，回过头来看看程序员编写"hello, world"时，
计算机内部究竟发生了什么。

探究止于内核，如果非要研究其中最本质的原理，一层一层追究下去，
还可以继续深入，到组成原理、数字电路、电子，甚至更加微观的结构，直到人类认知的极限。

限于能力和时间，没有针对每一个细节展开地很详细。
需要一定的C语言、操作系统、编译原理、计算机网络、Linux/shell基础。

# taocipian@x1-carbon ~ %
现在shell已经准备就绪，打印出一个"$"或者"%"或者其它符号作为提示符，
接着调用`fgets()`(或者其它类似函数)来读取用户输入的命令，
`fgets()`由runtime实现，可能还要调用若干其它函数，不过最终会到达`read()`，

这个`read()`(`man 2 read`)也是由runtime提供的，一般我们称它为"system call"，
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

可见这个伪"system call"是通过`int 0x80`陷入内核，调用内核里的`sys_read()`来交差的，后者才是真正的"system call"。

内核中，`sys_read()`会针对具体的对象调用不同的、具体的read函数，
例如read文件，则`read()`会接着调用`file_read()`；
如果read块设备，则`read()`会接着调用`block_read()`。

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

# % vi helloworld.c
  - 按键回显

用户按下一个键时，就会产生键盘中断请求信号，发送给中断请求控制器。
CPU在一个指令周期的最后时段内会检测中断请求，假设此时键盘中断优先级最高，CPU给予响应。

控制跳转到键盘中断处理程序`keyboard_interrupt`，
它做一些处理(例如将按下的键加入到读缓冲队列`read_q`中)，之后会调用`do_tty_interrupt()`，
`do_tty_interrupt()`没有其它任何操作，直接跳到`copy_to_cooked()`，
将读缓冲队列`read_q`中的数据按照[**cooked mode**](https://en.wikipedia.org/wiki/Cooked_mode)
进行处理后放进辅助队列`secondary`里，
在处理过程中，如果设置了回显（默认），则将该字符加入到写缓冲队列`write_q`中，
再调用`con_write()`将该字符显示在屏幕上。
处理完后，唤醒所有等待`secondary`队列的进程，其中就包括了shell。

键盘作为输入设备时，默认使用[行缓冲机制]()，此时，`copy_to_cooked()`遇到一个换行符或者EOF就会增加变量`tty->secondary.data`的值，以此记录输入缓冲中有几个输入行（行缓冲机制以行为单位交付给上层应用程序）。

`read_q`和`secondary`的关系。
`read_q`用于存储原始输入，`secondary`则存储处理后(cooked)的输入。
假设用户输入了"ABC<Backspace\>D"，
则`read_q`中为"ABC<Backspace\>D"，`secondary`中则为"ABD"。

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
用户接着输入，直到按下了一个回车，条件成熟，于是从`secondary`队列中将键入内容复制到用户缓冲区`buf`中。`tty_read()`在读取到需要的字节数或者行缓冲模式下读取到换行符就会返回。

这样，当用户输入`vi helloworld.c`并按下回车键后，`tty_read()`返回到内核中的`sys_read()`，接着中断返回到runtime的`read()`，一路向上直到`fgets()`返回，继续执行用户代码，此时输入的`vi helloworld.c`已经被shell获取。

  - 显示字符的原理
    - console
    - X(X的原理)
    - wayland

  - shell解释、执行

    - 对于用户输入的命令行（此处是`vi helloworld.c`），shell按一定流程进行处理，最终`fork()`出子进程执行`vi`，自己则等待`vi`的结束。
    - shell命令行处理步骤：[《shell脚本学习指南》](http://book.douban.com/subject/3519360)P179解释很详细。


  - file I/O

`vi`获取用户输入的字符，写入文件`helloworld.c`中。

# % gcc -Wall -g helloword.c
  - 编译：预处理、编译、汇编、链接

# % ./a.out
  - why `./`

一句话：安全考虑。

直接引用[PATH-wiki](https://en.wikipedia.org/wiki/PATH_(variable\))中的一段：

The current directory (`.`) is sometimes included by users as well, allowing programs residing in the current working directory to be executed directly. Superuser (root) accounts as a rule do not include it in `$PATH`, however, in order to prevent the accidental execution of scripts residing in the current directory, such as may be placed there by a malicious tarbomb.

  - 装载
  - 运行

# % gdb a.out
  - 调试器原理

===

参考资料

[Linux v0.11](https://www.kernel.org/pub/linux/kernel/Historic/old-versions/)

[《shell脚本学习指南》](http://book.douban.com/subject/3519360)

[《Linux内核完全剖析》](http://book.douban.com/subject/1481173/)

[《程序员的自我修养》](http://book.douban.com/subject/3652388/)
