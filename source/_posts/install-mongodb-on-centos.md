---
title: '在CentOS下安装MongoDB'
date: 2015-08-19 08:34:54
permalink: install-mongodb-on-centos
---

首先从官网下载最新版本的 mongodb（这里选择安装在`/usr/local`中）

```
cd /usr/local

wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.0.5.tgz
```

 解压缩：

```
tar -zxvf mongodb-linux-x86_64-3.0.5.tgz

mv mongodb-linux-x86_64-3.0.5 mongodb

rm -rf mongodb-linux-x86_64-3.0.5.tgz
```

 创建存放数据和日志的文件夹

```
cd mongodb

mkdir data

mkdir log

touch log/mongodb.log
```

 创建配置文件

```
vim mongod.conf
```

 加入以下内容：

```
fork = true   
port = 27017  
quiet = true  
dbpath = /home/mongodb/data  
logpath = /home/mongodb/log/mongodb.log  
logappend = true  
auth = false
```

 这些参数的意思是：

- `fork`：设置为`true`时启动后不会锁定命令行
- `port`：指定端口号
- `quiet`：设置为`true`为静默运行
- `dbpath`：指定数据的存放位置
- `logpath`：指定日志的存放位置
- `logappend`：设置为`true`时新日志会追加在文件后而不是覆盖掉文件
- `auth`：设置为`false`时不进行用户验证

 通过配置文件启动 MongoDB 服务端

```
/usr/local/mongodb/bin/mongd --config /usr/local/mongodb/mongod.conf
```

 使用客户端连接 mongodb

```
/usr/local/mongodb/bin/mongo
```

 创建一个通用的 admin 用户

```
use admin

db.createUser ({
    user: "admin",
    pwd: "password",
    roles: [
        {
            role: "userAdminAnyDatabase",
            db: "admin"
        }
    ]
})
```

 创建一个指定数据库的用户

```
use test

db.createUser ({
    user: "test",
    pwd: "test",
    roles: [
        {
            role: "userAdmin",
            db: "test"
        }
    ]
})
```

 停止 MongoDB 服务端

```
/usr/local/mongodb/bin/mongd --config /usr/local/mongodb/mongod.conf --shutdown
```

 编辑配置文件，将验证打开

```
vim mongod.conf

auth = true
```

 重新启动服务端

```
/usr/local/mongodb/bin/mongd --config /usr/local/mongodb/mongod.conf
```

 使用刚才创建的用户登陆

```
/usr/local/mongodb/bin/mongo -u admin -p password --authenticationDatabase admin
```

 到此为止 MongoDB 的安装配置便完成了，接下来将 mongod 注册为服务：

```
vim /etc/init.d/mongod
```

 添加以下内容：

```
#!/bin/bash  
#  
#chkconfig:2345 80 90  
#description:mongod

start () {  
 /usr/local/mongodb/bin/mongod --config /usr/local/mongodb/mongod.conf  
}  

stop () {  
 /usr/local/mongodb/bin/mongod --config /usr/local/mongodb/mongod.conf --shutdown  
}  

case "$1" in  
 start)  
start  
;;  

 stop)  
stop  
;;  

 restart)  
stop  
start  
;;  
 *)  
 echo  
$"Usage:$0{start|stop|restart}"  
 exit 1  
esac
```

 将其设置为可执行

```
chmod +x /etc/init.d/mongod
```

 添加服务

```
chkconfig --add mongodb
```

 设置开机启动

```
chkconfig MongoDB on
```

 之后便可以通过`service mongod start`、`service mongod stop`和`service mongod restart`命令对 MongoDB 服务端进行启动、停止和重启操作了。


