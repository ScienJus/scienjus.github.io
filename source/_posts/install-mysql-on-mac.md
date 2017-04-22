---
title: '在Mac下安装MySQL'
date: 2015-09-03 15:59:47
permalink: install-mysql-on-mac
---

最近开始将开发工具都转移到 Mac 上了，其中也会莫名其妙的遇到一些坑，不如干脆将整个流程都记录下来，方便以后查找。

### 下载与安装

 首先进入 [MySQL 官网](http://www.mysql.com/downloads/)，选择免费的`Community`版：`MySQL Community Server`。MySQL 官网提供了`tar.gz`和`dmg`两种格式的安装包，接下来主要围绕使用`dmg`安装来说。

![download](/uploads/2015/09/download.jpg)

 下载后双击打开安装包，根据说明一步步确认即可完成安装。安装成功后打开系统偏好设置，在最下面可以找到 MySQL，通过它可以查看 MySQL 状态并启动和关闭 MySQL。

![start](/uploads/2015/09/start.jpg)

### 指定配置

 使用该方式启动 MySQL 时是没有指定配置文件的，因此 MySQL 会按照以下顺序载入配置文件：

1. `/etc/my.cnf`
2. `/etc/mysql/my.cnf`
3. `/usr/local/mysql/etc/my.cnf`
4. `~/.my.cnf`

 只需要创建其中一个文件并设定好配置即可。

### mysql: command not found

 安装后直接在命令行输入`mysql`会提示`mysql: command not found`，需要先将其添加到环境变量：

```
vim ~/.bash_profile
```

 添加以下指令：

```
export PATH=${PATH}:/usr/local/mysql/bin
```

 保存后立即使其生效：

```
source ~/.bash_profile
```

 便可以使用 MySQL 的相关命令了。

### 配置用户

 刚安装好的 MySQL 只有一个密码为空的`root`用户，首先需要给该用户设置一个密码：

```
mysqladmin -u root password 你的密码
```

 然后使用设置好的密码登录 MySQl：

```
mysql -u root -p
 输入刚才设置的密码
```

 由于`root`用户只能用于本地登录，无法远程登录，所以还需要创建一个用于远程登录的用户：

```
GRANT ALL ON *.* TO 你的用户名 @'%' IDENTIFIED BY '你的密码' WITH GRANT OPTION;
```

 这条语句的意思是创建一个用户，赋予它以任何方式访问，在任何数据库的所有操作权限，但是实际这里的任何访问方式其实不包括`localhost`，所以还需要单独设定：

```
GRANT ALL ON *.* TO 你的用户名 @localhost IDENTIFIED BY '你的密码' WITH GRANT OPTION;
```

 与上一条语句一样，只是将任何访问方式改为了以`localhost`访问。

### ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (61)

 配置完远程登录的账户后，尝试使用该账户登录一下 MySQL：

```
mysql -h localhost -u 你的用户名 -p
 输入密码
```

 确实是可以登录成功的，可是如果使用 ip 登录：

```
mysql -h 127.0.0.1 -u 你的用户名 -p
 输入密码
```

 却会出现`ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (61)`的错误。原因是使用`dmg`安装的 MySQL 启动时的默认端口号不是`3306`而是`3307`，可以在这里修改：

```
sudo nano com.oracle.oss.mysql.mysqld.plist
```

 将`--port`的值改为`3306`即可：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key> <string>com.oracle.oss.mysql.mysqld</string>
    <key>ProcessType</key> <string>Interactive</string>
    <key>Disabled</key> <false/>
    <key>RunAtLoad</key> <true/>
    <key>KeepAlive</key> <true/>
    <key>SessionCreate</key> <true/>
    <key>LaunchOnlyOnce</key> <false/>
    <key>UserName</key> <string>_mysql</string>
    <key>GroupName</key> <string>_mysql</string>
    <key>ExitTimeOut</key> <integer>600</integer>
    <key>Program</key> <string>/usr/local/mysql/bin/mysqld</string>
    <key>ProgramArguments</key>
    <array>
      <string>/usr/local/mysql/bin/mysqld</string>
      <string>--user=_mysql</string>
      <string>--basedir=/usr/local/mysql</string>
      <string>--datadir=/usr/local/mysql/data</string>
      <string>--plugin-dir=/usr/local/mysql/lib/plugin</string>
      <string>--log-error=/usr/local/mysql/data/mysqld.local.err</string>
      <string>--pid-file=/usr/local/mysql/data/mysqld.local.pid</string>
      <string>--port=3307</string>
    </array>
    <key>WorkingDirectory</key> <string>/usr/local/mysql</string>
  </dict>
</plist>
```

 或是直接指定端口号连接：

```
mysql -h 127.0.0.1 -P 3307 -u 你的用户名 -p
 输入密码
```

### Communications link failure  
Out of resources when opening file '...'

 这两个都是 JDBC 连接 MySQL 时报的错，之所以放在一起说是因为改着改着莫名其妙两个都好了。

 配好 MySQL 项目后运行了一个多数据源并使用数据库连接池的项目，一共有 32 个数据源，tomcat-jdbc 连接池每个数据源初始化 10 个连接，在创建一半数据源之后开始出现第一个错误：

```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
```

 很多种情况都会导致这个错误，不过这里应该是数据库连接数不够，想到配置完后几乎没有编辑`my.cnf`文件，于是添加了这些属性：

```
max_connect_errors = 10000
max_connections=10000
max_user_connections=10000
```

 修改完后使用`show variables like 'max%';`查看数据库属性，发现对应数值确实改变了，但是运行程序还是会出现一样的问题。

 后来也 google 了很久，推测应该是系统配置导致的限制，但是也没找到问题具体出在哪里，只好先把初始化的线程数改小看看能不能正常运行，结果程序运行了一段时间后又出现另一个错误：

```
Out of resources when opening file './xxx.MYD' (Errcode: 24)
```

 这个错误的原因就很显而易见了，肯定是 MySQL 打开的文件过多了，但是我明明记得在`my.cnf`中设置了`open_files_limit`，不可能出现这个错误啊。

 结果使用`show variables like 'open%';`一看，配置竟然没生效，文件数只有可怜的 256，再三确认配置没有问题后认定原因肯定出在系统配置上。

 首先使用`ulimit -n`查看系统的文件数限制，果然是 256，看来 MySQL 会在`my.cnf`和`ulimit`中选择较小的一方作为真正的值。

 然后 google 一下 Mac 如何修改这个数值，一种方式是使用`launchctl limit maxfiles 65535`将文件数修改成 65535，尝试后发现确实系统的`ulimit -n`和 MySQL 的`open_files_limit`都变成了 65535。当时以为这样就解决了，结果又发现 JDBC 无法连接 MySQL 了，而 MySQL 客户端就能连上，真是匪夷所思。还好这个配置只要重启电脑就会失效，重启过后又能正常的连上 MySQL 了。

 还有一种方法只能用在 Mac OS X Yosemite 上的，首先创建`/Library/LaunchDaemons/limit.maxfiles.plist`文件，添加以下内容：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
      <string>limit.maxfiles</string>
    <key>ProgramArguments</key>
      <array>
        <string>launchctl</string>
        <string>limit</string>
        <string>maxfiles</string>
        <string>65536</string>
        <string>65536</string>
      </array>
    <key>RunAtLoad</key>
      <true/>
    <key>ServiceIPC</key>
      <false/>
  </dict>
</plist>
```

 创建`/Library/LaunchDaemons/limit.maxproc.plist`文件，添加以下内容：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple/DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
      <string>limit.maxproc</string>
    <key>ProgramArguments</key>
      <array>
        <string>launchctl</string>
        <string>limit</string>
        <string>maxproc</string>
        <string>2048</string>
        <string>2048</string>
      </array>
    <key>RunAtLoad</key>
      <true />
    <key>ServiceIPC</key>
      <false />
  </dict>
</plist>
```

 然后重启电脑就会发现数值已经改变了，所有方式也都能正常连接，我把初始化线程数调回来以后发现第一个错误也不会出现了。


