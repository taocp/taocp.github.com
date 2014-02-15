---
layout: post
description: "编码"
tweet-text: ""
author: taocp
tags:
- encoding
categories: 
- "unicode"
---

unicode:
=======
    一般指代标准，这个标准将一个字符和一个数字映射起来。这个数字用英语讲术就是code point，以U+开头，例如字符'A'对应的数字就是U+0061。它本身并不是一种编码方式。
utf-8：
=======
    是对unicode的一种实现，是一种编码方式。unicode标准只是规定了一个字符对应哪个数字，但没有规定一个字符要怎样存储。例如unicode标准规定字符'A'对应数字U+0061；但只要合适，可以用多种方法存储字符'A'（也就是存储数字0061），例如用4个字节存储'A'，依次为'0''0''6''1'。
    不同的实现方式(utf8 vs. utf16 vs. utf7 ...)各有优劣。目前utf-8是主流。

PS:在某些平台上（例如windows），unicode还可能指代utf-16编码。
