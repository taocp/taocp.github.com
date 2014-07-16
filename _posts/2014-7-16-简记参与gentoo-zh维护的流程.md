---
layout: post
description: "None.NULL"
tweet-text: ""
author: taocp
tags:
- gentoo
- opensource
categories:
- ""
---

偶尔也会给`gentoo-zh`提交几个ebuild，虽然都是从别的地方copy来的 >\_<

记录一下流程，供下次忘记时参考，也给可能需要的朋友一点提示吧。

如果您有更方便，更合理的方法，请联系我，谢谢。

## 创建本地overlay

github上fork `git@github.com:microcai/gentoo-zh.git` 一份到自己的仓库

把`gentoo-zh`拷贝到本地:`git clone git@github.com:yourname/gentoo-zh.git`

便于跟进`git remote add upstream git@github.com:microcai/gentoo-zh.git`

部分git && github的操作就省略了，诸如`git remote add origin ...`

在`/etc/portage/make.conf`中把刚才clone的overlay添加进来，如：
`PORTDIR_OVERLAY="/home/xxx/gentoo/gentoo-zh/  ${PORTDIR_OVERLAY}"`

## 添加ebuild并在本地测试

关于`Manifest`，团队成员的原话是`gentoo-zh uses thin-Manifest. We create Manifest for files need to fetch from the Internet, but not for ebuild itself.` 所以修改的时候注意点。

新建一个分支`git branch add-a-ebuild`

切换到新分支`git checkout add-a-ebuild`

添加ebuild文件，然后`ebuild xxx-version.ebuild digest`

先本地安装并测试一下`emerge -va xxx`

## push到github

确认无误后
`git add YYY`

`git commit -am "something"`

`git push origin add-a-ebuild`

然后在github上`compare & pull request`

等待`gentoo-zh`团队成员合并，可能需要修改才能通过。

成功合并后可以删除临时分支。

## 跟进，保持同上游的一致

不熟悉git，我是这么做的：

从上游获取最新的数据：`git fetch upstream master`

更新本地的数据：`git merge upstream/master master`

更新github上的数据：`git push origin master`

关于跟进，看到[这个方法](http://my.oschina.net/chbing/blog/198871)，下次试一试，再更新本文。
