---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- python
categories:
- python
---

PILLOW
-------

open a bmp file `Image.open(bmp_file)` , 


    got: error message like this : 
    raise IOError("Unsupported BMP header type (%d)" % len(s))`
    OSError: Unsupported BMP header type (108)`

该库目前还不支持所有BMP图像，
一个可能的简单方法，如果该BMP是由其它格式转换来的，尝试用GIMP转换/导出，导出为BMP时**勾选**不要写入xxx这个选项，保持兼容性。这个选项的英文原文是：`Do not write color space information`
