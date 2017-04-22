---
title: '使用Chrome Developer Tools调试线上网站'
date: 2016-04-02 08:44:56
permalink: chrome-developer-tools
---

起因是刷微博闲聊的时候发现评论的图片无法查看大图：

![微博](/uploads/2016/04/weibo1.jpg)

 本来以为是网的问题，但是当时闲得无聊，就随手打开了 Chrome Developer Tools (F12)，结果发现 Network 请求的图片地址很奇怪：

![Network](/uploads/2016/04/weibo2.jpg)

 本应该传一个图片 id 的地方却传了`null`和`undefined`，这说明肯定是请求之前取图片 id 失败了，查看一下 Elements 中的节点（直接在页面的对应位置右键，选择检查元素）：

![Dom 树](/uploads/2016/04/weibo3.jpg)

 可以看到`action-data`中的`pid`确实为`null`，这就是导致请求失败的直接原因，但是这里又是怎么出错的呢？

 一边观察 Elements 一边点击图片，可以发现这个元素并不是直接由服务端渲染而成的，而是通过 JS 动态加载的。那么就好说了，只需要查看一下加载的 JS 代码就可以了。

 在 Elements 中右键点击这个元素，然后选择 Break on -\\> Subtree Modifications，这样当这个元素发生改变时，便会自动断点到正在执行的 JS 代码。

![断点](/uploads/2016/04/weibo4.jpg)

 然后再次点击这个图片，触发 Dom 操作，发现断点停留在一坨被混淆和压缩的代码中。不过不要害怕，只需要点击下图左下角的`{}`按钮，JS 代码就会自动格式化了：

 格式化前：

![格式化前](/uploads/2016/04/weibo5.jpg)

 格式化后：

![格式化后](/uploads/2016/04/weibo6.jpg)

 接下来就是很普通的断点调试了，F8 是跳到下一个断点，F10 是下一步，F11 是进入方法，F12 是跳出方法。右侧列表还有调用栈，变量值等信息：

![调试](/uploads/2016/04/weibo7.jpg)

 虽然代码还是被混淆过，但已经足够排查出问题了：在下面这行代码中会通过一个正则表达式截取出图片的 id 赋值给`n`（也就是之后的`pid`）。但是由于某些原因图片链接与正则表达式并不匹配，导致没有截取到，所以最后`n`的值为`null`。

![Bug](/uploads/2016/04/weibo8.jpg)

 虽然这只是一个低级的 Bug，但是也可以轻描淡写的展示出 Chrome Developer Tools 的强大功能：监测网络请求、查看和动态修改 Dom 元素、格式化 JS 代码、断点调试运行中的 JS 代码。使你能够轻松的在开小差的时候一言不合就开始 Debug【误


