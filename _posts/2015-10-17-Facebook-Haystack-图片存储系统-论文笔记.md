---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- distributed
- storage
categories:
- ""
---

读了一下这篇论文，做一些笔记，主要是自己备忘、加深理解。

知识水平非常有限，可能会误导，有错误欢迎指出。

# Haystack与它解决的问题

小对象存储系统，高效解决长尾请求的检索问题：访问不那么热门照片的请求数量巨大，单纯cache搞不定。

存储的对象"written once, read often, never modified, and rarely deleted".

通过将metadata放在内存中，合理安排layout使得只有极少的请求会产生多于1次的磁盘操作（3.4.5 Filesystem）。

## 概念

- physical volume: A very large file(100 GB) saved as `/hay/haystack_<logical volume id>`.
Each volume holds millions of photos.

- logical volume: We further group physical volumes on different machines into logical volumes.

  我理解就是一个logical volume实际存储了N份physical volume做冗余保证数据安全。

- needle: a photo

- key: photo id

- alternate key: 同一张photo有4种不同不同尺寸，它们key相同，alternate key不同。（3.4 注释2）

## Haystack Directory

4个主要功能：

1. `map<logical volume, physical volume>`

1. load balances:
	- writes across logical volumes
	- reads across physical volumes


1. determines whether a photo request should be handled by the CDN or by the Cache

1. identifies those logical volumes that are read-only

参考3.1节，Directory还存储了application metadata，诸如每个photo所属的logical volumes等。
这个在构造url时需要。

## Haystack Cache

cache一个photo的条件比较有意思。

## Haystack Store

1. Each Store machine manages multiple physical volumes.

1. im-memory: `map<(key, alternate key), (needle's flags, size, volume offset)`

## 读

1. web servers通过Directory构造photo的url

   `http://<CDN>/<Cache>/<Machine id>/<Logical volume, Photo>`

   url中`Machine id`就是Store，读的时候负载均衡。

1. 浏览器根据url访问CDN或者Cache

1. Cache也不命中则访问Store，
   url中`Logical volume`可以找到Store上的`physical volumes`
   (某个大文件，其存储形式诸如`/hay/haystack_<logical volume id>`)，
   而用`Photo`信息可以找到该photo在`physical volumes`上的offset，seek一次读出数据

## 写

(可以参考Figure 4，这图距写这一章节略远，容易忽略)

1. web server从Directory请求一个write-enabled的logical volume
1. web server给photo分配一个唯一id并写到这个logical volume对应的每个physical volumes
1. Store收到web server的写请求（包含logical volume id, key, alternate key, cookie, data），
   同步地append到physical volume文件，并且更新内存映射

### 修改

Haystack不提供overwrite的功能，
修改图片通过“将修改后的图片以相同的key和alternate key写入”来实现。


## 删

1. Store在im-memory的map中设置删除标记
1. 同步设置volume文件中的标记位

# 疑问

1. Directory的挂掉后怎么处理？

    Directory貌似是个单点。

1. Directory的压力？

   看论文构造url的部分，并且考虑到负载均衡，貌似每次访问都需要Directory构造url，
那么一个Directory能抗住所有图片请求的压力？不科学。

还有很多细节问题。。。
