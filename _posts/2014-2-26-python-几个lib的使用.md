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

maybe a solution: convert format with GIMP and export to BMP with option **compatibility options** , checkout the `Do not write color space information`.(Chinese:导出为BMP时**勾选**不要写入xxx)
