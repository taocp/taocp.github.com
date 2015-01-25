---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- programming
- tools
categories:
- ""
---

gflags是google发布的一款命令行参数解析工具，简单好用。

[官方主页](https://code.google.com/p/gflags/) | [官方文档](https://google-gflags.googlecode.com/svn/trunk/doc/gflags.html)

安装gflags
===

以gentoo系统为例：`sudo emerge -va gflags`

如果没有root权限，也可以自己下载源码编译，安装到有权限的目录即可。

---
hello world
===

test-gflags.cc :

{% highlight cpp linenos %}
#include <iostream>
#include "flags.h"

int main(int argc, char * argv[])
{
    gflags::ParseCommandLineFlags(&argc, &argv, true);
    std::cout << FLAGS_ip << ":" << FLAGS_port << std::endl;
    return 0;
}
{% endhighlight %}

flags.cc :

{% highlight cpp linenos %}
#include <gflags/gflags.h>

DEFINE_int32(port, 0, "information of `port'");
DEFINE_string(ip, "0.0.0.0", "information of `ip'");
{% endhighlight %}

flags.h :
{% highlight cpp linenos %}
#ifndef FLAGS_H_

#include<gflags/gflags.h>

DECLARE_int32(port);
DECLARE_string(ip);

#endif
{% endhighlight %}

`g++ -Wall -lgflags test-gflags.cc flags.cc`

{% highlight bash %}
# 指定命令行参数时，风格比较自由
% ./a.out --port=403 --ip=192.168.1.1
192.168.1.1:403

# 这样也是可以的
% ./a.out -port 403 --ip 192.168.1.1
192.168.1.1:403
{% endhighlight %}

---
配置文件
===
如果参数选项较多，也可以放在专门的配置文件中，不用在命令行里直接指定。

gflags会自动根据 FLAGS_flagfile 的值打开配置文件读取各个选项。

例如：

test.flag :

```
--ip=127.0.0.1
--port=80
```

{% highlight bash %}
% ./a.out --flagfile=./test.flag
127.0.0.1:80
{% endhighlight %}

---
不要滥用
===
FLAGS_* 都是一些全局变量，所以应该慎重使用！笔者亲身体验，肺腑之言。 :(
