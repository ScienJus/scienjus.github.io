---
title: '博客迁移到 Github Pages 了'
date: 2017-04-23 16:11:51
permalink: moved-blog-to-github-pages
---

还有两个月专门放博客的主机就要到期了，仔细一想现在也懒得折腾 WordPress 了，干脆最后折腾一把弄成静态博客的丢到 Github Pages 吧！

<!--more-->

# 选型

这几年静态博客相关的方案已经有很多了，但是基本上都是基于 Markdown 去写文章，然后生成为 HTML 直接发布。

本来作为一个 Ruby 爱好者，再加上有 Github Pages 官方支持的加成，自然应该会去选择 Jekyll。但是无奈找了半天都找不到一款合适的主题，又懒得自己折腾，就选择了烂大街的 Hexo 和 [NexT][1]。这个主题功能很全面，配置起来很简单，非常适合我这种懒人。

关于如何安装 Hexo 以及相关配置就不在这里详述了，感兴趣可以直接去 [Hexo][2] 官网查看。

# 文章迁移

虽然我很早开始就通过 Markdown 写博客了，但是最开始在 WordPress 上找不到一个合适的 Markdown 渲染插件，一时脑残就选择了自己在本地渲染成 HTML 然后再通过 WordPress 发布，导致迁移的时候造成了不必要的麻烦。

我的文章迁移的流程为：

1. 通过 WordPress 的后台将所有博客的导出成 XML 文件
2. 将文章内容的 HTML 转换为 Markdown
3. 修改一些边角的转移字符
4. 按照文章链接输出到对应的文件

我通过一个 Ruby 脚本完成这一切，使用 `oga` 解析 XML，`reverse_markdown` 将 HTML 转为 Markdown，`auto-correct` 优化排版，整个脚本的代码大概如下（因为只是用一次所以写的比较随意）：

```ruby
require 'oga'
require 'reverse_markdown'
require 'time'
require 'auto-correct'

template = <<-TEMPLATE
---
title: '%s'
date: %s
permalink: %s
---

%s
TEMPLATE

handle = File.open('/path/to/wordpress.xml')

doc = Oga.parse_xml(handle)

doc.xpath('channel/item').each do |item|
  title = item.at_xpath('title').text
  time = item.at_xpath('pubDate').text
  link = item.at_xpath('link').text
  content = item.at_xpath('content:encoded').text
  # html 2 markdown
  content = ReverseMarkdown.convert(content, github_flavored: true).inspect
  # auto correct
  content = content.auto_correct!

  File.open("/path/to/blog/#{link}.md", 'w').write(template % [title, time, link, content])
end
```

需要注意的是在将 HTML 转成 Markdown 后可能会出现一些边边角角的转义字符，上面的程序并没有列出这部分逻辑，需要自己观察博客并修改。

# 图片迁移

之前的图片是直接上传到 WordPress 资源库，用七牛云做 CDN，如果当初直接把图片丢到七牛也就没这破事了，但是我这次依旧没有把图片丢到七牛（主要就是懒，应该也不会再迁移第三回了）。

个人比较建议将所有图片都放在 `source` 下，也就是和 `_posts` 平级，这样当使用 MWeb 这样对其有支持的编辑器时，可以比较好的提供本地预览的支持

![mweb](/uploads/mweb.png)

Hexo 官方推荐的另外一种做法是设置 `post_asset_folder: true` 然后将每个文章使用的资源都放在与文章同名的文件夹下。但是由于我需要将以前的图片迁移过来，这样做只会增加额外的迁移工作量。

而且这种做法据说还会带来图片在首页或归档页无法正常显示的问题，以至于兼容这个问题还需要在 Markdown 里人为的添加 Hexo 独有的标签，这太不清真了，果断拒绝。

# 评论迁移

在最开始就用了 Disqus，之后依旧会使用它，所以整个迁移过程很简单。

Disqus 是通过 URL 识别每篇文章的，官方也提供了非常丰富的迁移工具，如果只是域名发生了更改可以直接使用 [Domain Migration Tool][3]，如果文章的地址也发生了改变就需要使用 [URL Mapper][4] 了。

在博客完全替换之前，为了预览效果我给新博客分配了一个二级域名，而当我在博客迁移完、将主域名转到新博客后，却发现评论没有如预期的那样同步过来。

此时我在 Disqus 上正常评论是可以显示的，看后台导出的记录发现新评论依旧是挂在二级域名下，最后将所有评论都从主域名转到二级域名下，评论终于能够在新博客上默认显示了。

所以在此也建议在博客完全迁移完之前，不要进行 Disqus 相关的配置。

# 版本管理

在文章的最初曾经提到 Github Pages 默认支持的引擎是 Jekyll，这意味着如果你使用 Jekyll 写博客，只需要将作为源文件的 Markdown 发布到 Github，就能自动跑一套 CI 生成静态页面，整个版本管理会简单很多。

很遗憾的是 Hexo 目前并不具备这种条件，这意味着只能先在本地生成静态页面，再将静态页面发布到 Github 上，而源文件没有任何版本控制，只有本地存储了一份。

目前我在同一仓库中又建立了一个 `source` 分支，每次写好博客后，先会通过普通 Git 操作流程将源文件发布到该分支，然后再通过 `hexo g -d` 将最新的页面发布到 `master` 分支。也有朋友建议我提交源文件后通过第三方 CI 进行发布，但是我觉得写博客作为一个低频行为（至少对于我来说是低频行为），不值得搞得这么复杂。

----

最后，虽然还有很多东西没有弄（统计、计数、CDN），但是至少可以开始产出文章了，愿这个新平台给我带来更良好、更高效的写作体验。


[1]: https://github.com/iissnan/hexo-theme-next
[2]: https://hexo.io/zh-cn/docs/
[3]: https://help.disqus.com/customer/portal/articles/912627-domain-migration-wizard
[4]: https://help.disqus.com/customer/portal/articles/912757-url-mapper


