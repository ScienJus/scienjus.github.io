---
title: '使用wekan作为看板工具'
date: 2016-03-12 11:11:49
permalink: wekan
---

前段时间，为了解决团队内部的沟通问题，我开始寻找一种能够减少沟通成本，并且简单易用的团队协作工具，这时候发现了这种名为「看板」的工具。

 在这里并不想详细去介绍看板到底是个什么东西，以及它对于团队开发到底有着怎样的作用。事实上最后还是没能在团队中推广起来，这说明仅仅拥有好的工具是不够的，没有对应的管理以及沟通能力一样无法解决问题。

 虽然在团队内使用的计划是搁浅了，但我自己却在了解后对这类应用产生了兴趣。在此之前我习惯将所有琐事都记录在印象笔记中，包括一些计划、想法、博客素材、代码片段等。印象笔记很适合用来存储博客素材和代码片段，前者将会在积累到一定程度后用使用 Markdown 重新排版并发布，而后者配合搜索在工作中也能很好的提升效率。但是印象笔记却无法直观的展现想法和计划，导致可能仅仅是记录后就忘掉了，再也没派上用场。

 对于我而言，这时候引入看板就可以将这些想法和计划根据不同优先级和类型整理汇总排出 TODO List，并且还可以记录我每个月（或是每周）具体做了哪些事情，坚持一段时间后再看想必也会有一些成就感。也许这并不是看板最正确的用法，但这确实很符合我的需求。

 既然决定了尝试一番，就需要具体选择一款看板工具了，看上去 Trello 应该是最方便的选择，但是奔着瞎折腾的精神还是要选择一款开源的产品，于是找到了 [wekan](https://github.com/wekan/wekan)。

![wekan 的界面](/uploads/2016/03/wekan.jpeg)

 看板应用的功能大体都差不多，选择 wekan 的原因有两个：一个是之前写`qqbot`的时候，在 GitHub 上关注了维护另一个版本的开发者`@floatinghotpot`，那时候他正好也在贡献 wekan 的代码，所以第一反应就想到了 wekan，第二个原因是 wekan 的开发者也在使用自己开发的这款软件进行着团队协作，并且公开在 [网上](https://wekan.io/b/MeSsFJaSqeuo9M6bs/wekan-roadmap)，这点实在是令人敬佩。

 如果你不确定 wekan 是否能够满足你的需求，你也可以在直接在 [这里](https://oasis.sandstorm.io/appdemo/m86q05rdvj14yvn78ghaxynqz7u2svw6rnttptxx49g1785cdv1h) 试用一下，如果它正好合你的胃口，那就可以开始安装了。

wekan 的安装有三种方式，使用 Docker 安装、通过 Sandstorm 安装和通过 GitHub 上的 Releases 包手工安装（实际还可以直接 Fork 源码编译安装，实在没必要），这里我选择的是第三种方式。

### 安装 Node.js

 作为一个 JS 盲，我首先直接在官网下载了最新的 4.4 版本，然后编译安装一切顺利。但是等到使用`npm`安装 wekan 依赖时，却发现无法安装`fibers`。上网搜了一下，原来必须要降级到`0.10.40`版本才能正确安装，所以这里无法直接在官网安装最新的版本，但是还有以下几种方案：

1. 如果你使用的是 CentOS，直接使用`yum`安装就可以了，版本正好是`0.10.40`
2. 先安装 Node.js 的版本控制工具`nvm`，然后使用它安装对应版本的 NodeJS
3. 在 [这里](http://nodejs.org/en/blog/release/v0.10.40/) 下载对应操作系统的 NodeJS，然后编译安装

### 安装 MongoDB

wekan 使用 MongoDB 作为数据库，而 MongoDB 的安装也非常简单，只需要在官网下载对应的包，解压后写个配置文件就可以启动了。如果你不太了解的话也可以看 [这篇文章](http://www.scienjus.com/install-mongodb-on-centos/)

### 安装 Meteor

Meteor 的安装十分简单，只需要一行命令：

```
curl https://install.meteor.com/ | sh
```

### 安装 wekan

 首先从 [Releases](https://github.com/wekan/wekan/releases) 中下载最新的版本，并解压

```
wget https://github.com/wekan/wekan/releases/download/v0.10.1/wekan-0.10.1.tar.gz
tar zxvf wekan-0.10.1.tar.gz
mv wekan-0.10.1.tar.gz wekan
```

 如果你解压出来的直接就是`bundle`文件夹，那么就自己建一个`wekan`文件夹并移动进去吧。

 进入`wekan/bundle/programs/server`安装：

```
cd wekan/bundle/programs/server && sudo npm install
```

 配置环境变量：

```
export MONGO_URL='mongodb://127.0.0.1:27017/wekan'
export ROOT_URL='https://example.com'
export MAIL_URL='smtp://user:pass@mailserver.example.com:25/'
export PORT=8080
```

 这里我不太确定 wekan 是否支持 MongoDB 开启鉴权，Wiki 和 Issues 上都没有找到结果。

 返回到`wekan/bundle/`启动服务：

```
cd ../../
node main.js
```

 此时启动如果没有报错，并且通过浏览器访问对应的地址可以看到 wekan 的主页就是配置成功了。

### 配置 Nginx

 由于 Wekan 使用了 WebSocket，所以如果使用 Nginx 映射的话需要开启相关配置，例如：

```
server {
    listen 80;
    server_name wekan.scienjus.com;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

 之后就可以开始使用 wekan 并享受它给你带来一切都井井有条的舒爽体验了。我索性直接将自己的 TODO List 直接放在了 [这里](http://wekan.scienjus.com)，即使一个人使用难免有些寂寞，像是成员、评论等功能也都用不到。但这已经比之前使用印象笔记要好的多了！


