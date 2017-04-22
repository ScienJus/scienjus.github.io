---
title: 'Web QQ协议分析（六）：其他'
date: 2015-12-19 13:58:54
permalink: webqq-analysis-6
---

### 获取最近会话列表

 请求方式：`Post`

url：`http://d1.web2.qq.com/channel/get_recent_list2`

 请求参数依旧只有一个`r`，值为 JSON，内容为：

```
{
  vfwebqq: "\#{vfwebqq}",
  clientid: 53999199,
  psessionid: "\#{psessionid}"
}
```

 这里的`psessionid`传一个空字符串也是可以的，但是最好还是传了，没准哪一天就校验了呢。

 请求成功后会返回一个非常简陋的 JSON：

```
{
    "result": [
        {
            "type": 1,
            "uin": 2856977416
        }
    ],
    "retcode": 0
}
```

 返回数据只提供了`type`和`uin`两个字段，分别表示会话的类型和编号，`type`的值为`0`时表示这是一个好友会话，`uin`就是对方的编号，`type`为`1`时表示这是一个群会话，`uin`为群的编号，`type`为`2`时表示是讨论组会话，`uin`为讨论组编号。

 从这里可以明显的看出，Web QQ 的服务端倾向于只返回必要的数据，由客户端在本地保存大量的信息和对应的关系。这样做可能主要还是为了性能做打算，但是对接起来真是麻烦爆了，我仅仅是想将消息打印到 Console，还要在接收到消息后调用多个接口去查群信息、群成员信息、QQ号等等，实在是麻烦爆了。

### 获取当前登录用户信息

 请求方式：`Get`

url：`http://s.web2.qq.com/api/get_self_info2&t=0.1`

referer：`http://s.web2.qq.com/proxy.html?v=20130916001&callback=1&id=1`

 返回内容和 [获取好友详细信息](http://www.scienjus.com/webqq-analysis-4/) 一模一样。


