---
title: '数据库水平分片心得'
date: 2015-09-21 11:03:08
permalink: database-sharding-review
---

### 什么是数据库的水平分片

 水平分片（也叫水平分库）指的是将整体存储在单个数据库中的数据，通过某种策略分摊到多个表结构与其相同的数据库中，这样每个数据库中的数据量就会相对减少很多，并且可以部署在不同物理服务器上，理论上能够实现数据库的无限横向拓展。所以水平分片是数据库性能问题的最终解决方案。

### 什么时候需要水平分片

 在考虑这个问题之前，首先先列举一下数据库的升级方案：

 在开发初期，为了快速实现功能，一般会将所有数据存放在单个数据库上。

 单个数据库最大的缺点是一旦该数据库由于某些原因挂掉，整个软件的所有功能都面临瘫痪。所以为了保证数据库的高可用，通常会再加一个数据库与其形成主从关系（Master-Slave），此时所有操作还是由主数据库完成，主数据库再同步到从数据库上，而从数据库只需要在主数据库挂掉之后代替其工作。也就是说此时的从数据库只是为了容灾，对性能没有任何提升。

 当遇到第一次数据库性能问题时，最先想到的方案应该是读写分离，将所有写操作都放在主数据库上，所有读操作都放在从数据库上，这样至少可以将主数据库的压力减半，并且由于大部分软件都是读多写少，所以可以配置一主多从进一步减少数据库的读压力。

 一般在这时候还会加入缓存（使用 Memcached 或 Redis），与数据库本身没有太大关系就不细说了，一般来说读写分离加上缓存已经可以应付绝大多数情况了，并且几乎不需要对业务层面进行修改。如果再次遇到性能问题，则需要根据业务逻辑进行选择更进一步的方案了，一般有以下几种：

 对数据库进行垂直分库，将业务彼此无关的表放在单独的数据库中，分库后不同库中的表无法进行联合查询等操作，但是可以平摊压力，并且独立做读写分离。

 对数据库进行水平分表，建立多个结构相同的表分摊数据，使得每个表的数据量减少从而提升速度。分表和最开始提到的数据库分片看起来很像，实际各有优劣：分表后各个子表还是可以通过 union 等命令联合查询，分库后则不行。但是分库后每个数据库可以独立部署在不同的服务器上，充分利用多台服务器的性能，分表却只能在单台机器的单个数据库上，如果是服务器本身的性能达到瓶颈，则分表不会有明显作用。

### 水平分片策略

 最开始提到了，既然要将单个数据库中的数据分摊到多个数据库，自然不能随便乱分，而是要选择合适的策略。常见的分库策略有这些：

1. 基于 id 的区间分片，例如：将 id 为 1-2w 的数据存放在 A 数据库，2w-4w 的数据存放在 B 数据库。缺点是单库能承受的数据量需要预估，如果预估的不准确容易造成性能不够用或者浪费。
2. 基于 id 的 hash 分片，例如：将 id%2=0 的数据存放在 A 数据库，id%2=1 的数据放在 B 数据库。缺点是在动态增加数据库时，hash 的结果会发生变化，所以需要对已有数据进行迁移，一般是用一致性 hash 或在分库初期就建立足够的数据库避免这个问题。
3. 基于时间的区间分片，大部分软件都会有一个特征：越新的数据被操作的几率越大，老数据几乎不会被操作。所以通过数据的插入时间进行分库（也称为冷热分离），例如：将 2015年1月-5月 的数据放在一个库，6月 到 12月 的数据放在另一个库。这样做的缺点是在查询时需要额外提供数据的创建时间才能找到数据存放在哪个库，所以比较适合微博等主要以时间轴（Timeline）功能为主的软件。
4. 基于检索表分片，通过额外建立一张检索表保存 id 与所在数据库节点的对应关系，优点是逻辑简单，自由且不会有迁移问题，缺点是每次查询都需要额外查询检索表，所以一般会选择将检索表缓存到内存中。
5. 基于地理位置分片，像点评、滴滴打车之类的软件由于不同城市的数据不需要互通，可以按照城市分片，将不同城市的数据存放在不同数据库中，这样做的一个优点是可以将数据库服务器部署到离对应城市最近的节点上，以提高访问速度。

 这些分库策略都有各自的优点和缺点，需要根据业务选择最合适的策略，我现在体验的第二种应该是最常见的一种了，所以接下来主要介绍一下这种策略会发生哪些问题，并且该如何解决。

### 基于 id 的 hash 分片遇到的问题

**遍历表**

 曾经最简单的操作在分库后成为了第一个难题。由于数据分成了多份，遍历表就需要查询多次数据库。这个还比较好解决，使用多线程查询的话也不会慢太多。但是分页就很难解决了，例如：

 有一张微博表，它的结构如下：

```
create table weibo (
    id int primary key auto_increment comment '主键',
    user_id int comment '发微博的用户 id',
    create_at datetime comment '创建时间',
    ...其他字段
) comment '微博';
```

 单库情况下查询当前最新 20 条微博的 SQL 语句为：

```
select * from weibo order by create_at desc limit 20;
```

 分库情况下的 SQL 语句为：

```
use db_1;
select * from weibo order by create_at desc limit 20;

use db_2;
select * from weibo order by create_at desc limit 20;

use db_3;
select * from weibo order by create_at desc limit 20;
....
```

 需要将所有库的最新 20 条都查询出来排序后再取出总的最新 20 条。

 如果改成需要查询当前最新的第 20-40 条微博的话，单库的 SQL 语句为：

```
select * from weibo order by create_at desc limit 20, 20;
```

 分库的 SQL 语句为：

```
use db_1;
select * from weibo order by create_at desc limit 40;

use db_2;
select * from weibo order by create_at desc limit 40;

use db_3;
select * from weibo order by create_at desc limit 40;
....
```

 注意这里取出的不是 20-40 条了，而是最新的 40 条，因为整体的第 20-40 条微博在每个子库中只能保证排在前 40，却不能保证肯定在 20-40 这个区间，这样查询的数据量就翻倍了。以此类推，如果需要查询第 15000 条-15020 条微博的话，查询的效率肯定是无法忍受的。

 解决办法是在额外建立一张索引表，保存 id 和排序、筛选字段，先从索引表中查询对应的 id，再通过 id 去查询具体信息。索引表如果不是很复杂的话，完全可以通过 Redis 的 zset 和 list 实现。

**主键冲突**

 在数据库表设计时，为了保证 id 唯一，大部分人都会将主键设为自增的 int 类型。但是由于 auto\\_increment 是和表所绑定的，所以在分库后每个表的自增 id 也是独立的。这样就肯定会发生主键冲突，解决办法有以下几种：

 最简单的方法是使用公用的 id 生成器，比如在 MySQL 中建立一张公共的表用于生成 id，或者通过 Redis 的 incr 生成 id。

 如果已经预先分配好了数据库的个数，并且不会改变，可以直接对每个库的所有 auto\\_increment 设定步长和初始值。例如一开始就分配了 36 个数据库：

```
SET @@auto_increment_increment=36;

use db_1;
alter table weibo auto_increment=1;

use db_2;
alter table weibo auto_increment=2;

use db_3;
alter table weibo auto_increment=3;
....
```

 这样 db\\_1 的自增 id 就会变成 1, 37, 73…，db\\_2 的自增 id 就会变成 2, 38, 74…，只要保证不会改变数据库的个数，id 就肯定不会冲突。

 如果只需要主键唯一，而不一定是自增整数的话，也可以使用 uuid，它会生成 36 位（去掉"-"是 32 位）的字符串，并且几乎不会重复。

 但是很多人都希望主键即使不是连续自增，也是一个有序的整数，这样在排序等情况下会有用。这时候就需要自己实现一个 id 生成算法了，一般都会使用 unix 时间戳保持有序，混入 Mac 地址等保证唯一。

**禁止联合查询**

 假设有一张用户表和一张关注关系表，它们的结构分别是：

```
create table user (
    id int primary key auto_increment comment '主键',
    sex tinyint comment '性别:0 女,1 男'
    ...其他字段
) comment '用户';

create table follow (
    user_id int comment '用户 id',
    follow_id int comment '关注的用户 id',
    ...其他字段
) comment '关注关系';
```

 查询用户 1 都关注了谁的 SQL 语句为：

```
select * from user u inner join follow f on f.follow_id = u.user_id where f.user_id = 1;
```

 但是在分库情况下这样就行不通了，因为不是所有用户都和用户 1 在同一个数据库中，而这样的联合查询只能查出与用户 1 在同一个数据库的用户，所以 SQL 语句需要改写为：

```
ids = 
select follow_id from follow where user_id = 1;

use db_1;
select * from user where id in (ids);

use db_2;
select * from user where id in (ids);

use db_3;
select * from user where id in (ids);
....
```

 在分库情况下，需要将大部分联合查询都替换为至少两次查询，先从关联的表中查询出符合条件的 id，在根据 id 去对应的数据库里查询主体信息。

**关联表的冗余字段**

 假设刚才的需求变更了，需要查询用户 1 关注的所有男性用户，并且以每页 20 条进行分页，单库的 SQL 语句需要变为：

```
select * from user u inner join follow f on f.follow_id = u.user_id where f.user_id = 1 and u.sex = 1 limit 20;
```

 多库的 SQL 语句变为：

```
ids = 
select follow_id from follow where user_id = 1 limit 20;

use db_1;
select * from user where id in (ids) amd sex = 1;

use db_2;
select * from user where id in (ids) amd sex = 1;

use db_3;
select * from user where id in (ids) amd sex = 1;
....
```

 这样看上去没什么问题，但是最后查询出来的数据量却达不到分页量。原因是正常逻辑中筛选应该是在分页之前的，但是这里却是先分页后筛选，所以才会出现数据量不够的情况。

 解决办法是在 follow 表中冗余 user 的 sex 字段，直接在第一次查询的时候就完成筛选和分页。但是这样就需要在 user 的 sex 字段发生变更时动态更新 follow 表中的 sex 的字段。

**关联表的冗余表**

 单库情况下用户 1 关注了用户 2 的 SQL 语句为：

```
insert into follow (user_id, follow_id) values (1, 2);
```

 查询用户 1 都关注了谁的 SQL 语句为：

```
select * from user u inner join follow f on f.follow_id = u.user_id where f.user_id = 1;
```

 查询用户 2 都被谁关注了的 SQL 语句为：

```
select * from user u inner join follow f on f.user_id = u.user_id where f.follow_id = 1;
```

 而如果在多库中，查询 db\\_1 的用户 1 都关注了谁的 SQL 语句为：

```
select follow_id from db_1.follow where user_id = 1;
```

 查询 db\\_2 中的用户 2 都被谁关注了的 SQL 语句为：

```
select user_id from db_2.follow where follow_id = 2;
```

 那么如果用户 1 关注了用户 2，这条 follow 表中的 insert 记录该插入到哪个数据库中呢，如果插入到 db\\_1，用户 2 则查不到用户 1 关注了他，如果插入到 db\\_2，用户 1 则查询不到关注了用户 2，所以只能在两个数据库都插入一条数据。

 对于表之间的关联关系，尽量将一对多，多对一中的关联数据放在同一个分片下（例如一个用户和他所发的所有微博哦），多对多关系最坏情况需要在在关联双方所在的数据库都存放一条记录，并且保持这多条记录的同步更新。

**分布式事务**

 一个业务操作可能会有多个数据库操作，所以需要事务来保证这些操作的完整性，但是如果这些操作对应的是多个不同的数据库，就比较麻烦了。我对分布式事务的了解也不是很深，这里就不详细介绍了。

### 总结

 数据库水平分片作为数据库性能瓶颈的最终解决方案，确实可以有效的解决这个问题。但是它将业务逻辑变得非常复杂（主要是关联表冗余和字段冗余，以及这些数据的更新），并且有分布式事务这个难题。所以不到必要时刻，尽量不要轻易尝试数据库分片。而是先去选择对业务影响较小的读写分离，垂直分库等方案。


