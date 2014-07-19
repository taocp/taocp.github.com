---
layout: post
description: "编码"
tweet-text: ""
author: taocp
tags:
- encoding
categories: 
- None 
---

unicode将一个字符和一个数字(code point)映射起来。例如字符`a`对应的数字就是`U+0061`，汉字`哈`对应就是`U+54c8`[参考](http://www.chi2ko.com/tool/CJK.htm)。以`U+`开头，表unicode。

unicode只是规定了字符和`code point`的映射关系，只有最单纯善良的人才会真就把每个字符存储为它的`code point`4个字节。 因为这样严重浪费存储空间(也会有兼容性的问题)，例如每个英文字符都由原来的ASCII码1字节上升为4字节。

回忆huffman编码，通过赋予高频对象短编码，低频对象长编码来实现压缩效果。 同理，可以在实际存储时赋予高频字符尽量短的编码来节省存储空间。这样的编码方式不止一个，其中之一就是`utf-8`。在Vim中，`g8`可以参看光标所在字符的`uft-8`编码。

所以，`unicode`是抽象的概念和标准，而`utf-8`则是实际的实现方案。
