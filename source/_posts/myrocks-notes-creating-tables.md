---
title: MyRocks 学习笔记之创建表
date: 2019-07-21 21:03:19
tags:
permalink: myrocks-notes-creating-tables
---

## 前言

目前网上介绍 MyRocks 的文章虽然不少，但是大部分都只介绍了一些 RocksDB 的核心特性和读写原理，却几乎不会提到 MyRocks 在实现 MySQL 存储引擎相关的内容，并且由于 MySQL 官方对于存储引擎的开发资料也提供的很单薄，所以对于新人来说难免有些手足无措。

这个系列希望通过从 MySQL 存储引擎的 API 作为起点，结合 MyRocks 的实现，记录下每一个功能的全貌，包括自定义的存储引擎在每一个 API 中具体需要实现哪些功能，以及 MyRocks 是如何通过 RocksDB 实现这些功能的，其优缺点是什么。希望能够帮助一些初学者（包括我自己）如何从零开始或是二次开发一个 MySQL 存储引擎。

这篇笔记是第一章，介绍了创建表（Creating Tables）的流程。

<!--more-->

## 接口定义

官方文档的 [Creating Tables](https://dev.mysql.com/doc/internals/en/creating-tables.html) 章节简要的介绍了自定义的存储引擎如何实现创建表的功能，只需要实现 `create` 这个虚函数即可。

```
virtual int create(const char *name, TABLE *form, HA_CREATE_INFO *info)=0;
```

存储引擎需要在这个函数中创建所有与表结构和索引结构相关的数据文件，它有三个参数：

- `name`: 该表的表名
- `form`: 该表的元数据信息，主要包含表结构、字段和索引的信息
- `info`: 创建表时的额外的配置信息，基本都是 `CREATE TABLE` 时附带的选项

## MyRocks 实现

MyRocks 的实现在 `ha_rocksdb.cc` 的 `ha_rocksdb::create` 方法中。主要逻辑分为两部分：

1. 对用户提交的信息做一些转换和校验，拒绝 MyRocks 存储引擎不支持的配置
2. 按照 MyRocks 的存储方式重新组织表结构和索引结构，并存储在内存和 RocksDB 中


### 前置处理

MyRocks 首先会对创建表的配置信息进行前置处理，包括配置的检查和转换，拦截该存储引擎不支持的配置等，主要流程为：

1. `DATA DIRECTORY` 和 `INDEX DIRECTORY` 支持将该表的数据文件和索引文件存放在一个指定的路径。MyRocks 不支持这两个配置，而是通过 `rocksdb_datadir` 配置 RocksDB 存放数据的地址。
2. 参数中的表名格式为 `./$dbname/$tablename`，MyRocks 会将其格式化为 `$dbname.$tablename`，便于之后处理。
3. 解析 SQL 语句中是否含有外键定义，MyRocks 不支持外键，如果含有外键也会返回错误。


接下来还需要检查当前这个表是否已经存在了，在 `TRUNCATE TABLE` 语句下需要删除重名的表信息，其他情况下报错。

```
Rdb_tbl_def *tbl = ddl_manager.find(str);
if (tbl != nullptr) {
  if (thd->lex->sql_command == SQLCOM_TRUNCATE) {
    err = delete_table(tbl);
    if (err != HA_EXIT_SUCCESS) {
      DBUG_RETURN(err);
    }
  } else {
    my_error(ER_METADATA_INCONSISTENCY, MYF(0), str.c_str(), name);
    DBUG_RETURN(HA_ERR_ROCKSDB_CORRUPT_DATA);
  }
}
```

其中包含两个细节：

什么时候会出现 `CREATE TABLE` 到存储引擎时，ddl_manager 中已经有了表的数据，却没有被上层拦截？

在[这个 Issue ](https://github.com/facebook/mysql-5.6/issues/37)中提到了一个场景，即 frm 文件丢失（例如被人工删除）的情况，会进入该逻辑，需要做容错处理。

为什么需要判断 `sql_command == SQLCOM_TRUNCATE`，什么场景会出现？

通过看 `sql_truncate.cc` 中的逻辑猜测，如果存储引擎支持通过重建表实现 `TRUNCATE TABLE` 功能，那么上层会直接通过 `create` 方法创建一个结构完全相同的空表，而不是通过存储引擎实现的 `truncate` 方法。

```
bool hton_can_recreate;

if (lock_table(thd, table_ref, &hton_can_recreate))
  DBUG_RETURN(TRUE);

if (hton_can_recreate)
{
  /*
    The storage engine can truncate the table by creating an
    empty table with the same structure.
  */
  error= dd_recreate_table(thd, table_ref->db, table_ref->table_name);

  if (thd->locked_tables_mode && thd->locked_tables_list.reopen_tables(thd))
      thd->locked_tables_list.unlink_all_closed_tables(thd, NULL, 0);

  /* No need to binlog a failed truncate-by-recreate. */
  binlog_stmt= !error;
}
```

并且 MyRocks 是支持 `HTON_CAN_RECREATE` 功能的。

```
rocksdb_hton->flags = HTON_TEMPORARY_NOT_SUPPORTED |
                      HTON_SUPPORTS_EXTENDED_KEYS | HTON_CAN_RECREATE;
```

所以需要考虑到这种情况，删除当前该表的数据并继续执行创建流程。


### 创建表和索引

创建表和索引的主要流程也就是将表结构以及索引结构存储到硬盘的流程。其中 `ddl_manager` 对象就是 MyRocks 中对 RocksDB 操作的封装。顾名思义，这个类只负责 DDL 相关操作的存储。

#### 开启事务

```
const std::unique_ptr<rocksdb::WriteBatch> wb = dict_manager.begin();
```

WriteBatch 是 RocksDB 中原子操作和批量操作的封装类。之后所有对 RocksDB 的写入操作都将写入到该 WriteBatch 中，这样可以保证这些操作可以合并成一个原子操作提交到 RocksDB 中，不会出现一部分逻辑报错导致数据不一致的情况。


#### 设置隐藏主键

```
/*
  If no primary key found, create a hidden PK and place it inside table
  definition
*/
if (has_hidden_pk(table_arg)) {
  n_keys += 1;
  // reset hidden pk id
  // the starting valid value for hidden pk is 1
  m_tbl_def->m_hidden_pk_val = 1;
}
```

MyRocks 支持表不设置主键，但是 RocksDB 底层的 KV 存储强依赖表的主键，所以在这里会自动增加隐藏主键列，并对上层透明。

#### 检查索引规范

```
/* MyRocks supports only the following collations for indexed columns */
static const std::set<const my_core::CHARSET_INFO *> RDB_INDEX_COLLATIONS = {
    &my_charset_bin, &my_charset_utf8_bin, &my_charset_latin1_bin};

static bool rdb_is_index_collation_supported(
    const my_core::Field *const field) {
  const my_core::enum_field_types type = field->real_type();
  /* Handle [VAR](CHAR|BINARY) or TEXT|BLOB */
  if (type == MYSQL_TYPE_VARCHAR || type == MYSQL_TYPE_STRING ||
      type == MYSQL_TYPE_BLOB) {
    return RDB_INDEX_COLLATIONS.find(field->charset()) !=
           RDB_INDEX_COLLATIONS.end();
  }
  return true;
}
```

当索引字段为 `varchar/string/blob` 等字符类型时，MyRocks 只支持编码为 `binary/utf8_bin/latin1_bin`。

通过关闭 `rocksdb-strict-collat​​ion-check` 或是在 `rocksdb-strict-collat​​ion-exceptions` 配置表名可以跳过这个检查。

#### 创建 Column Family

在 RocksDB 中，每一个 KV 都会关联一个列族（Column Family，之后简称为 CF），而 MyRocks 是以索引为粒度存储 KV 数据的，所以支持为每个索引配置一个可选的 CF，默认存放在 `default` 中。

CF 的名称可以通过索引的整个注释内容或是 `cfname=$name` 选项进行配置，例如：

```
CREATE TABLE sample (
    id INT PRIMARY KEY AUTO_INCREMENT,
    uid INT,
    name VARCHAR(25),
    ts TIMESTAMP,
    KEY(`uid`) COMMENT 'cfname=cf_uid',
    KEY(`name`) COMMENT 'cf_name'
) ENGINE=ROCKSDB
```

其中 id 的主键索引会关联到 `default` CF 中，uid 的索引会关联到 `cf_uid` 中，而 name 的索引会关联到 `cf_name` 中。

在代码中的实现逻辑很简单，只是遍历每个索引，通过注释截取出 CF 的值。
CF 不能是 `__system__`，这个 CF 已经预留给了存放系统的数据，包括之后将会存放表结构和索引结构的数据。

在之前的版本中，还可以通过 `cfname=$per_index_cf` 自动生成格式为 `$tablename.$indexname` 的名称，但是在最新版本的代码中已经不支持了。

光从建表的流程中我们还不知道索引的 CF 具体的用途是什么，会在之后的写入数据的文章再详细介绍。

#### 读取 TTL 数据

因为 RocksDB 本身支持 TTL，所以 MyRocks 也支持在建表时设置每一条记录的 TTL 选项，通过表级别的注释 `ttl_duration=1;ttl_col=ts` 进行设置。

#### 生成索引 ID

`Rdb_ddl_manager` 在内存中维护了一个自增的索引 id，启动时会从本地 RocksDB 中读取并初始化。当需要创建索引时，会通过调用 `get_and_update_next_number` 方法申请一个 id。其会在内存中加锁自增后写入 RocksDB，其格式为：

- Key：`Rdb_key_def::MAX_INDEX_ID`
- Value：`Rdb_key_def::MAX_INDEX_ID_VERSION, val`


#### 初始化自增起始值

如果建表时指定自增主键的初始值 `auto_increment`，MyRocks 则会将其写入 system CF 中，格式为：

- Key：`Rdb_key_def::AUTO_INC, cf_id, index_id`
- Value：`Rdb_key_def::AUTO_INCREMENT_VERSION, auto_increment_value`

这里通过 RocksDB 的 merge operator 实现了更高性能的自增操作，不过建表时肯定是初始化，所以语义应该和 Put 相同。


#### 写入 CF Flags

目前 MyRocks 有两个 CF 级别的配置，需要额外存储到以 CF 为单位的数据中，被称为 CF Flags，包括：

1. `is_per_partition_cf` 表示这个 CF 是否为某个分区表特定的 CF，例如配置 `p0_cfname=cf_p0`。
2. `is_reverse_cf` 表示这个 CF 中存储的数据是否要反向存储，这样会使降序查询（`order by desc`）更快，配置方法是 `cfname=rev:xxx`。

这两个 flag 分别占用 1bit，最终会合并保存在 RocksDB 中，格式为：

- Key：`Rdb_key_def::CF_DEFINITION, cf_id`
- Value：`Rdb_key_def::CF_DEFINITION_VERSION, flags`

因为多个索引可以共享同一个 CF，所以需要保证索引在创建时，CF 的配置不能和之前索引的冲突。


#### 写入索引信息

在 `Rdb_dict_manager::add_or_update_index_cf_mapping` 方法中，会将每一个 Index 的信息存储在 RocksDB 中。

- Key：`Rdb_key_def::INDEX_INFO, cf_id, index_id`
- Value：`Rdb_key_def::INDEX_INFO_VERSION_LATEST, index_type, kv_version, index_flags, ttl_duration`


#### 写入表和索引的映射关系

在 `Rdb_tbl_def::put_dict` 中，会将一个表所对应的 CF 和 Index 存储到 RocksDB 中。

- Key：`Rdb_key_def::DDL_ENTRY_INDEX_START_NUMBER, db_table_name`
- Value：`DDL_ENTRY_INDEX_VERSION, cf_id1, index_id1[, cf_id2, index_id2...]`


#### 缓存表信息

之后 `Rdb_ddl_manager::put` 方法会将这些信息同样缓存在内存中，便于之后其他操作使用，主要存放了两部分数据：

1. 在 `m_ddl_map` 中缓存 `db_table_name` 对应的 `Rdb_tbl_def`
2. 在 `m_index_num_to_keydef` 中缓存 `index_id, cf_id` 和 `db_table_name, index_no` 的映射关系


#### 提交事务

在所有操作处理完后，`Rdb_dict_manager::commit` 方法将以上所有的改动都通过 WriteBatch 提交到 RocksDB 中，同最开始提到的一样，这是一个原子操作，只会全部成功或是全部失败。


## 实际验证

光看代码肯定会存在遗漏或是理解错误的地方，接下来让我们实际创建一张表验证一下。

以下是用来测试的表，一共有四个字段、一个主键索引和两个普通索引，并且通过注释指定了 CF 和自增起始值。

```
CREATE TABLE sample (
    id INT PRIMARY KEY AUTO_INCREMENT,
    uid INT,
    name VARCHAR(25),
    ts TIMESTAMP,
    KEY(`uid`) COMMENT 'cfname=cf_uid',
    KEY(`name`) COMMENT 'cf_name'
) ENGINE=ROCKSDB AUTO_INCREMENT=100;
```

创建成功后，我们查看一下存储在系统表中的 DDL 信息。

```
select * from INFORMATION_SCHEMA.ROCKSDB_DDL;
```

![](/uploads/15637654866041.jpg)

接下来查看 RocksDB 当中的数据，并与上面的 DDL 信息以及代码分析进行比对。

```
sst_dump --command=scan --file='000038.sst' --output_hex
from [] to []
Process /var/lib/mysql/#rocksdb/000038.sst
Sst file format: block-based
'00000001746573742E73616D706C65' seq:16, type:1 => 0001000000000000010000000002000001010000000300000102
'000000020000000000000100' seq:11, type:1 => 000601000D000000000000000000000000
'000000020000000200000101' seq:13, type:1 => 000602000D000000000000000000000000
'000000020000000300000102' seq:15, type:1 => 000602000D000000000000000000000000
'0000000300000000' seq:6, type:1 => 000100000000
'0000000300000001' seq:5, type:1 => 000100000000
'0000000300000002' seq:12, type:1 => 000100000000
'0000000300000003' seq:14, type:1 => 000100000000
'00000007' seq:9, type:1 => 000100000102
'000000090000000000000100' seq:10, type:2 => 00010000000000000064
```

第一条数据为表和索引的对应关系：

```
00000001               => 常量 DDL_ENTRY_INDEX_START_NUMBER
746573742E73616D706C65 => 'test.sample'，即库表名

0001                   => 常量 DDL_ENTRY_INDEX_VERSION
00000000               => 0，即 PRIMARY 的 cf id
00000100               => 256，即 PRIMARY 的 index number
00000002               => 2，即 uid 的 cf id
00000101               => 257，即 uid 的 index number
00000003               => 3，即 name 的 cf id
00000102               => 258，即 name 的 index number
```

第二条到第四条数据为索引信息，以第二条为例：

```
00000002               => 常量 INDEX_INFO
00000000               => 0，即 PRIMARY 的 cf id
00000100               => 256，即 PRIMARY 的 index number

0006                   => 常量 INDEX_INFO_VERSION_LATEST
01                     => index type，1 是主键索引，2 是索引，3 是隐藏索引
000D                   => kv format version，primary 和 secondary 都是 13
00000000               => index flags，目前只有 TTL_FLAG
0000000000000000       => ttl duration
```

第五条到第八条数据为 CF 信息，以第五条为例：

```
00000003               => 常量 CF_DEFINITION
00000000               => cf 的 id

0001                   => 常量 CF_DEFINITION_VERSION
00000000               => cf flags，因为既没有分区也没有逆序，所以为 0
```

第九条数据为当前系统内最大的索引 id：

```
00000007               => 常量 MAX_INDEX_ID

0001                   => 常量 MAX_INDEX_ID_VERSION
00000102               => 当前系统最大的索引 id 是 258
```

第十条数据是自增数据：

```
00000009               => 常量 AUTO_INC
00000000               => 0，即 PRIMARY 的 cf id
00000100               => 256，即 PRIMARY 的 index number

0001                   => 常量 AUTO_INCREMENT_VERSION
0000000000000064       => 100，即设置的初始自增值
```


## 参考资料

1. [myrocks记录格式分析](https://yq.aliyun.com/articles/62648)
2. [MySQL · myrocks · 相关tools介绍](http://mysql.taobao.org/monthly/2017/12/10/)
3. [Schema Design](https://github.com/facebook/mysql-5.6/wiki/Schema-Design)
4. [Column Families on Partitioned Tables](https://github.com/facebook/mysql-5.6/wiki/Column-Families-on-Partitioned-Tables)
5. [MyRocks Information Schema](https://github.com/facebook/mysql-5.6/wiki/MyRocks-Information-Schema)
6. [MySQL · MyRocks · TTL特性介绍](http://mysql.taobao.org/monthly/2018/04/04/)
7. [MySQL · RocksDB · Column Family介绍](http://mysql.taobao.org/monthly/2018/06/09/)
8. [MySQL · myrocks · collation 限制](http://mysql.taobao.org/monthly/2018/09/09/)


