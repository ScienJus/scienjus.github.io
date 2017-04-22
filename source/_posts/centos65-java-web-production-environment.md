---
title: 'CentOS 6.5搭建Java Web生产环境'
date: 2015-09-15 03:49:13
permalink: centos65-java-web-production-environment
---

公司项目终于决定从托管机房搬到阿里云了，再也不用因为机房抽风加班了！

 不太重要的软件就直接用`yum`安装了（例如 VsFtpd 之类的就不在本文介绍了），但是像 Nginx、MySQL 等最好还是下载最新稳定版本的源码编译安装。

### JDK

 在 Oracle 下载 jdk 的 rpm 安装包，这里下载的是 1.7 版本：

```
wget http://download.oracle.com/otn-pub/java/jdk/7u79-b15/jdk-7u79-linux-x64.rpm?AuthParam=1444378409_f275580f53339110b974a239bd3f4379
mv jdk-7u79-linux-x64.rpm?AuthParam=1444378409_f275580f53339110b974a239bd3f4379 jdk-7u79-linux-x64.rpm
rpm -ivh *.rpm
```

jdk 会默认安装在`/usr/java/`目录下，接着配置环境变量：

```
vim /etc/profile

JAVA_HOME=/usr/java/jdk1.7.0_79
CLASSPATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib
PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
export PATH CLASSPATH JAVA_HOME
```

 配置好以后测试一下：

```
java -version
```

 出现以下信息说明配置好了：

```
java version "1.7.0_79"
Java (TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot (TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
```

### Tomcat

 首先下载并解压 tomcat（个人习惯会把 tomcat 放在`/usr/local/`下）：

```
wget http://mirror.bit.edu.cn/apache/tomcat/tomcat-7/v7.0.64/bin/apache-tomcat-7.0.64.tar.gz
tar -zxvf apache-tomcat-7.0.64.tar.gz
mv apache-tomcat-7.0.64 tomcat
```

 配置 Web 项目（个人习惯通过`tomcat/conf/Catalina/localhost/`下编写 xml 配置文件）：

```
vim website.xml
```

 记得指定 Tomcat 的编码，否则会出现接收参数乱码：

```
vim conf/server.xml
```

 启动一下试试：

```
/usr/local/tomcat/bin/startup.sh
```

### MySQL

 下载：

```
wget http://cdn.mysql.com/Downloads/MySQL-5.6/mysql-5.6.27.tar.gz
```

 准备好编译的库：

```
yum -y install make gcc-c++ cmake bison-devel ncurses-devel
```

5.6 版本开始通过`cmake`编译：

```
tar -zxvf mysql-5.6.27.tar.gz
cd mysql-5.6.27
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/usr/local/mysql/data -DSYSCONFDIR=/etc -DWITH_MYISAM_STORAGE_ENGINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_MEMORY_STORAGE_ENGINE=1 -DWITH_READLINE=1 -DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock -DMYSQL_TCP_PORT=3306 -DENABLED_LOCAL_INFILE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DEXTRA_CHARSETS=all -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
make && make install
```

mysql 不建议直接通过 root 用户开启数据库，所以最好创建一个用户和组：

```
groupadd mysql
useradd -g mysql mysql
chown -R mysql:mysql /usr/local/mysql
```

 初始化：

```
/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --user=mysql
```

 注册开机自启动：

```
cp support-files/mysql.server /etc/init.d/mysql
chkconfig MySQL on
```

 初始化 root 用户的密码：

```
/usr/local/mysql/bin/mysqladmin -u root password '你的密码'
```

 关于 MySQL 的配置在这里就不详细介绍了。

### Redis

 根据 Redis 官网的介绍，安装只需要三步：下载、解压、编译：

```
wget http://download.redis.io/releases/redis-3.0.4.tar.gz
tar -zxvf redis-3.0.4.tar.gz
mv redis-3.0.4 redis
cd redis
make
```

 启动方式：

```
redis-server redis.conf
```

redis 不需要配置用户，在配置文件中设置一下密码就可以了，关于配置同样不详细介绍了。

### MongoDB

 这个更省事，都不需要编译，下载完解压就可以用了：

```
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.0.6.tgz
tar -zxvf mongodb-linux-x86_64-3.0.6.tgz
mv mongodb-linux-x86_64-3.0.6 mongodb
```

 启动方式：

```
mongod -f mongodb.conf
```

mongodb 默认是不需要验证权限的，如果需要验证权限的话，需要在添加一个用户，并且在配置文件中设置`auth = true`，例如：

```
use database;
db.createUser (
  {
    "user" : "username",
    "pwd": "password",
    "roles" : [
      {
        role: "dbOwner",
        db: "database"
      }
    ]
  }
);
```

 连接方式：

```
mongo -u username -p password --authenticationDatabase database
```

### Nginx

 使用 nginx 是为了代理一些静态文件，同时做一下多 tomcat 实例的负载均衡，不过静态文件我还没弄（好像阿里云 OSS 就能弄？），先说一下负载均衡吧。

nginx 的 yum 包也比较旧了，依旧选择源码编译，首先下载：

```
wget http://nginx.org/download/nginx-1.8.0.tar.gz
tar -zxvf nginx-1.8.0.tar.gz
cd nginx-1.8.0
```

 指定一下安装路径，编译安装：

```
./configure --prefix=/usr/local/nginx
make && make install
```

 负载均衡的配置事例：

```
vim nginx.conf

include proxy.conf;
upstream localhost {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
        listen 80;
        server_name localhost;

        #charset koi8-r;

        #access_log logs/host.access.log main

        location / {
            proxy_connect_timeout 3;
            proxy_send_timeout 30;
            proxy_read_timeout 30;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://localhost;
        }
}

vim proxy.conf

proxy_redirect off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X_Forwarded-For $proxy_add_x_forwarded_for;
client_max_body_size 10m;
client_body_buffer_size 128k;
proxy_connect_timeout 300;
proxy_send_timeout 300;
proxy_read_timeout 300;
proxy_buffer_size 4k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k;
proxy_temp_file_write_size 64k;
```

 一个基础的 Java Web 项目的生产环境边搭建完成了。


