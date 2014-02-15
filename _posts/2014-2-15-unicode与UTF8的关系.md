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
    一般指代标准，这个标准将一个字符和一个数字映射起来。这个*数字*用英语讲术说是code point，以U+开头，例如字符'A'对应的数字就是U+0061。它本身并不是一种编码方式。
utf-8：
    是对unicode的一种实现，是一种编码方式。假设我们新创一种编码方式utf-789，每个字符都用4个字节表示，编码(encoding)字符'A'就表示为'0''0''6''1'。编码U+8631对应的字符就用4个字节'8''6''3''1'来表示依次类推。不同的实现方式各有优劣。目前utf-8是主流。
    PS：在某些平台上（例如windows），unicode还可能指代utf-16编码。
