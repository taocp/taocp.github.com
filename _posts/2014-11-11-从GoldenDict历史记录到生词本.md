---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- programming
- linux
categories:
- programming
---

[GoldenDict](http://goldendict.org)是目前用过的Linux下最好的一款词典，支持发音、划屏取词。
合理配置还能识别出单词的各种时态（菜单Edit -- Dictionaries -- Sources -- Morphology）。

美中不足是它的历史记录功能还不完善，只能按查询的实际顺序记录20个单词，
没有给用户提供任何操作接口，所以没办法将查询的生词直接加入生词本。

这里通过监听历史记录文件`$HOME/.goldendict/history`内容的改动来获取查询的单词，
然后将这个单词追加到生词本里。

---

## 监听

通过Linux内核提供的`inotify(7)`可以把监听任务交给系统，在有相关事件发生时通知应用程序，
从而避免应用程序的轮询或忙等，提高效率。

简单来说，应用程序对一个特殊的文件描述符执行`read()`从而阻塞，事件发生时`read()`返回，
应用程序处理相应事件。

关于`inotify`:

1. `man 7 inotify`

1. [Kernel Korner - Intro to inotify](http://www.linuxjournal.com/article/8478)

---

## 实现

奇怪的是，通过`inotify`监测，发现每次发生查询时，被修改的文件是`$HOME/.goldendict/history.tmp`，
而不是`history`。暂时没有精力去研究goldendict的源码，就不追究啦。

所以，`inotfiy`在这里不是直接监听history文件，而是监听`$HOME/.goldendict/`目录，
监测到events再判断对应的文件是否为history.tmp，是则执行添加生词的操作，否则忽略该事件。

运行：`./a.out ~/.goldendict`

{% highlight c linenos %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <assert.h>
#include <sys/types.h>
#include <sys/inotify.h>
#include <linux/limits.h>   // PATH_MAX, NAME_MAX

/*
 * http://www.linuxjournal.com/article/8478
 */

#define EVENT_SIZE  (sizeof(struct inotify_event))
#define	EVENTS_ONCE 10          /* 至少支持一次性10个events，此处够用了 */
#define BUF_LEN     (EVENTS_ONCE * (sizeof(struct inotify_event) + PATH_MAX + 1))
#define	WORD_MAX    25          /* 单词的最大长度 */
#define	CMD_MAX     128			/* shell命令的最大长度 */

void add_new_word(void);

int main(int argc, char *argv[])
{
    if(argc != 2){
        printf("usage: ./%s  directory\n", argv[0]);
        exit(1);
    }
    int fd = inotify_init();
    assert(fd != -1);
    int wd = inotify_add_watch(fd, argv[1], IN_MODIFY);
    assert(wd != -1);

    char buf[BUF_LEN];
    char *fname = "history.tmp";
    int len;
    while( (len=read(fd, buf, BUF_LEN)) > 0 ){
        int i=0;
        while(i<len){
            struct inotify_event *event = (struct inotify_event*)(buf+i);
            if(event->mask & IN_MODIFY){
                printf("the file/directory %s modified\n", event->name);
                if(!strncmp(event->name, fname, NAME_MAX)){
                    add_new_word();
                }
            }
            else{
                printf("%d\n", event->mask);
                // ignore these events
            }
            i += EVENT_SIZE + event->len;
        }
    }
    inotify_rm_watch(fd, wd);
    close(fd);
    return 0;
}

/*
 * 从历史记录文件中读取最新的一个单词，并将它加入生词本。
 *
 * 我这里，单词本是一个普通的文本文件，
 * shell命令`e`是一个自定义的shell脚本，
 *  % e a_new_word
 * 执行的操作就是将新单词a_new_word追加到单词本里。
 * 之所以调用system()执行额外的shell命令，而不是fopen()操作：
 * 一是有了这个shell命令，需要时也可以在终端里添加生词到单词本；
 * 二是既然本程序的核心利用了Linux内核的特性，也就不必要考虑基于windows的可移植性啦。
 *
 * 根据"单词本"的实际情况修改。
 */
void add_new_word(void)
{
    char *history = "/path/to/.goldendict/history" // correct path && add the ';'
    char word[WORD_MAX];
    char cmd[CMD_MAX];
    FILE *his = fopen(history, "r");
    assert(his);


    if( 1 != fscanf(his, "%*s %s", word)){// history文件第二列才是单词
        return ; // 读取history出错
    }
    printf("word:%s\n", word);
    fclose(his);

    sprintf(cmd, "$HOME/bin/e %s\n", word);
    system(cmd);
    printf("cmd:%s\n", cmd);
}

#undef  EVENT_SIZE
#undef	EVENTS_ONCE
#undef  BUF_LEN
#undef	WORD_MAX
{% endhighlight %}

---

## 不足

- 判断/清理

程序无法检测一个查询是否真的适合被加入生词本，它采取“总是加入”的策略，
由用户定期对生词本进行手动清理，权当复习。

- 太多啦，不一一列举。。。
