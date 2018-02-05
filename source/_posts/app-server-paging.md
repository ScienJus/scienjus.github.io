---
title: 'APP后端分页设计'
date: 2015-07-29 10:21:52
permalink: app-server-paging
---

### 移动端套用传统分页的缺点

 目前数据分页一般分为两种类型：传统网站比较常见的电梯式分页布局及移动端比较常见的流式分页布局。

 电梯式分页布局在传统网站中非常常见，比如百度、淘宝：

![baidu](/uploads/2015/07/baidu.png)

![taobao](/uploads/2015/07/taobao.png)

 它的特点是在网站的底部有分页栏，用户不仅可以点击上一页、下一页浏览数据，还可以直接点击页码跳转到特定页，所以电梯式分页的的 SQL 查询（以下称为传统分页）也比较统一，基本上为前端提供页数及每页的数量，后端套用下面的 SQL 查询语句：

```
#currentPage 为当前页数（以 1 开始），pagingSize 为每页的数据量
select * from ... where ... order by ... limit (currentPage- 1) * pagingSize, pagingSize;
```

 流式分页布局在移动端比较流行，因为移动端的屏幕尺寸普遍较小，会导致分页栏不容易点击。并且移动端拥有良好的滑动体验，向上滑动加载更多，向下滑动刷新的操作方式更加便利。

![weibo](/uploads/2015/07/weibo.png)

 在后端的处理上，传统分页逻辑是完全可以套用到流式分页布局上的。并且因为不用提供总页数或总数据量还能减少查询次数，但是也会暴露出如下问题：

#### 数据重复

![repeat](/uploads/2015/07/repeat.png)

 比如：

![smzdm](/uploads/2015/07/smzdm.png)

#### 数据缺失

![lose](/uploads/2015/07/lose.png)

#### 效率低

 使用 limit 在数据量小的时候并不会有效率问题，但是当数据偏移量很大时性能会开始急剧下降，查询性能比对会在接下来提到。

 综上所述，流式分页不需要也不适合使用传统分页，而是使用一种更合适的方法：游标分页。

### 游标分页举例

 假设有一个`news_`表用来存放新闻信息，这个表的结构如下：

```
# 新闻表，这个表可能还有一些别的字段，不过与主题无关就暂时省略掉了
create table news_ (
    id_ int auto_increment comment 'id',
    title_ varchar (200) comment '标题',    
    content_ text comment '正文',
    create_date datetime comment '创建日期',
    ...        
    primary key (id_)
);
```

 众所周知，大部分的软件都会把最新的新闻放在最前面，这里我们假设刨除其他因素，单纯以`create_date`倒序排列，使用传统分页的 SQL 语句如下：

```
#currentPage 为当前页码（以 1 开始），pagingSize 为每页的数据量
select * from news_ order by create_date desc limit ($currentPage- 1) * $pagingSize, $pagingSize;
```

 而游标分页则不需要提供当前页码，而是提供当前页的起始位置（也称为游标）用于定位，游标分页的 SQL 语句如下：

```
#cursor 为上一页最后一条新闻的 create_date（如果是第一页则为当前时间），pagingSize 为每页的数据量
select * from news_ where create_date > $cursor order by create_date desc limit $pagingSize
```

 从这里可以看出，传统分页的偏移量是固定的，所以会因为数据的新增或减少导致接下来加载数据重复或丢失。而游标分页则不会出现这种情况，因为当数据发生新增和减少时，游标的位置也会相对变化。

### 效率对比

 之前提到传统分页在偏移量大时性能会急剧下降，而游标分页则不会，在这里同样以`news_`表做个实验。

```
# 创建新闻表
drop table if exists news_;
create table news_ (
    id_ int auto_increment comment 'id',
    title_ varchar (200) comment '标题',
    content_ text comment '正文',
    create_date datetime comment '创建日期',
    primary key (id_)
);

# 自动添加数据的存储过程
create procedure insert_to_news ()
begin
    declare i int;
    set i = 0;
    while i <= 1000000 do
        insert into news_(title_, content_, create_date) values('title', 'content', random_date());
        set i=i+1;
    end while;
    commit;
end
```

 使用上面的 SQL 语句创建一个拥有 100w 条数据的`news_`表，分别使用传统分页和游标分页测试一下查询效率：

```
# 查询时间:0.928s
select * from news_ order by create_date desc limit 900000, 20;

# 这里的 1911-05-01 20:06:57 是我生成数据中与上边查询对应的游标，请自行替换。
# 查询时间:0.146s
select * from news_ where create_date > '1911-05-01 20:06:57' order by create_date desc limit 20;
```

 可以看出在偏移量较大的情况下，即使没有添加索引游标分页的查询时间都超越传统分页几倍，而在添加了索引之后几乎不会因为偏移量造成任何影响。

### 游标的另一种用法，更新新增数据

 向新浪微博等软件，当在主页面放置一段时间后，就会收到提醒`有 1 条新微博，点击查看`，如果用户点击，则会将新增的数据添加到顶端。很明显，这样的操作也不是传统分页能够实现的，所以大部分使用传统分页逻辑的程序在面对此需求时会直接清除所有内容，重新加载第一页。

![news](/uploads/2015/07/news.png)

 很显然这并不是一个很好的方法，因为很有可能新增的数据只有 1、2 条，而为了这点数据刷新整页是非常浪费的。此时如果使用游标则会轻松很多，同样以`news_`表举例：

```
# 获得进入页面后新增的新闻数，cursor 为当前页面的第一条新闻的创建时间
select count (id_) from news_ where create_date > $cursor;

# 如果新增的新闻数较小，点击则只加载新增的新闻，cursor 为当前页面的第一条新闻的创建时间
select * from news_ where create_date > $cursor order by create_date desc;
```

 这样便可以在新增量比较小的时候直接加载新增数据放在页面顶部，在新增量较大时才选择重新加载第一页数据。

### 总结

 传统分页的特点

- 可以直接根据页码跳转到特定页
- 可能会出现重复、丢失数据的情况
- 页数较大时性能会降低
- 排序条件与分页无关

 游标分页的特点

- 不可以直接跳转到特定页，只能加载下一页
- 不会出现重复、丢失数据的情况
- 查询效率与页数无关，并且优于传统分页
- 不适合排序条件比较复杂的分页

