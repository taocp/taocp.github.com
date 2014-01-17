---
layout: post
description: "从windows转到linux的新手可能遇到的问题"
author: taocp
categories:linux
---

&nbsp;&nbsp;&nbsp;&nbsp;记录一些windows与linux双系统导致的问题，主要是windows给linux带来的问题 ^Q^

  * linux下程序可执行属性丢失。
由于windows和linux下主流的文件系统的差异，windows下的文件系统并不记录文件的可执行权限。所以linux分区下有可执行权限的程序一旦放到windows分区下，“可执行”这个属性就会丢失。例如，linux下一些软件的源码包不要放在windows分区下编译，否则可能会产生这样的错误：configure: error: cannot run C compiled programs.If you meant to cross compile, use `--host'. 当然，对于某些可执行脚本，实在需要可以bash your-script.sh来执行。
