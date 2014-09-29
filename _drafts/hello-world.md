"hello, world"

**Linux v0.11 kernel为例**

"hello, world"是我编程生涯中的第一个程序，
读了3年大学，回过头来看看程序员编写"hello, world"时，
计算机内部究竟发生了什么。

探究止于内核，如果非要研究其中最本质的原理，一层一层追究下去，
还可以继续深入，到组成原理、数字电路、电子，甚至更加微观的结构，直到人类认知的极限。

# taocipian@thinkpad ~ %
现在shell已经准备就绪，打印出一个"$"或者"%"或者其它符号作为提示符，
接着调用`fgets()`(或者其它类似函数)来读取用户输入的命令，
`fgets()`由runtime实现，可能还要调用若干其它函数，不过最终会到达`read()`，

这个`read()`(man 2 read)也是由runtime提供的，一般我们称它们为"system calls"，
但其实这个`read()`不是真正的"system calls"，看看一个简单的`read()`实现。

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

可见这个伪"system calls"是通过`int 0x80`陷入内核，调用内核里的`read()`来交差的。
内核中，`read()`会针对具体的对象调用不同的、具体的read函数，
例如read文件，则`read()`会接着调用`file_read()`；
如果read块设备，则`read()`会接着调用`block_read()`；
此时shell是等待键盘输入，于是`read()`会接着调用`rw_char()`，
而`rw_char()`经过层层调用，最终会跳到`tty_read()`，
`tty_read()`会把tty设备输入队列中的数据复制到用户缓冲区buf中，
也就是`int read(int fd, void *buf, int count)`中的那个`buf`。
幸运的是，此时tty设备输入队列为空，没有内容可以复制，只能等待用户输入，
由于在CPU看来等待用户输入是一个非常缓慢的过程，
于是为了节约时间进程(内核态)主动sleep让出CPU，内核会在某个恰当时机唤醒shell进程。


# vi helloworld.c
  - 按键回显
  - shell获取键入内容
  - 显示字符的原理
    - console
    - X(X的原理)
    - wayland
  - shell解释、执行
  - file I/O

# gcc -Wall -g helloword.c
  - 编译：预处理、编译、汇编、链接

# ./a.out
  - why "./"
  - 装载
  - 运行

# gdb a.out
  - 调试器原理

===

涉及知识

  - 操作系统(调度、内存、I/O、FS)
  - 编译原理
  - 计算机网络(X)
  - C语言
  - 调试原理
  - Linux
  - shell
  - (数据结构与算法)
