---
title: '使用迅雷下载百度盘文件'
date: 2015-10-06 06:00:59
permalink: download-from-baiduyun-by-thunder
---

现在很多地方都流行将文件保管在百度盘。但是百度盘一旦下载比较大的文件，就必须要安装百度云管家，而且百度云管家还有限速，确实比较恶心。所以在这里介绍一下如何绕过这个拦截直接用迅雷下载大文件。

 首先登录百度盘，将需要下载的文件保存到自己的云盘中。

![保存文件](/uploads/2015/11/pc.png)

 然后在浏览器中输入`http://pan.baidu.com/wap/`。这个地址是百度盘手机端的个人主页，如果你能成功进入这个地址，并且没有跳转到 PC 端主页的话。恭喜你，直接找到刚才保存的文件，点击下载即可。会自动触发浏览器的下载。

 但是很可惜的是，这个方法存活了不到一个月就被百度禁掉了，如今进入`http://pan.baidu.com/wap/`后，无论是 PC 端还是手机端都会自动跳转到`http://pan.baidu.com/disk/home`。原因是下面这段 JS 代码：

```
reg3=/wap\\/home/i;

jumptourl = url.match (reg3) ? url.replace (reg3, "disk/home");

window.location.href = jumptourl
```

 既然知道是这段 JS 代码在作祟，最简单的方法自然是直接将浏览器设置为禁止执行 JS 代码，例如 Chrome 的设置方法为：

```
 设置 -> 显示高级设置 -> 内容设置 -> 不允许任何网站运行 JavaScript
```

![Chrome 设置禁止执行 JS 代码](/uploads/2015/11/chrome.jpg)

 保存后重新打开`http://pan.baidu.com/wap/`，会发现没有再跳转到 PC 端主页。

![手机端主页](/uploads/2015/11/mobile.png)

 找到之前保存的文件，点击进入下载页面，恢复允许执行 JS 代码，刷新后点击下载按钮就可以跳转到浏览器下载了。

![手机端下载页面](/uploads/2015/11/download.png)


