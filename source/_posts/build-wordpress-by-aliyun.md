---
title: 阿里云搭建 WordPress 全过程
date: 2015-07-01 11:36:35
permalink: build-wordpress-by-aliyun
---

经过一上午的努力，博客终于搭建好了，在此将整体流程介绍一下，供各位参考。

# 域名注册

拥有一个独立域名是建立个人博客中重要的一环，注册域名可以选择 GoDaddy 或者万网。GoDaddy 经常会有优惠活动，可以以很便宜的价格买到理想的域名。可是我注册时并没有赶上，于是就在万网上注册了首年 39 元的 .com 域名，域名注册成功后暂时放置，等到购买完服务器后才会用到。

# 购买服务器

服务器的地域是最先需要考虑的问题，国外的服务器在大陆访问速度可能会慢一些，但是会有一个很大的优点就是域名不需要备案，而国内的服务器虽然访问速度较快但域名备案会比较麻烦(由于我选择了阿里云，所以也在痛苦备案中)。 这里简单的介绍一下阿里云主机的购买过程：

1.  打开[阿里云官网](http://www.aliyun.com/)，注册用户并登陆后点击购买云服务器ECS进入选择配置页面。
2.  如果是Linux系统的话不需要选择太好的配置，这里我选择的是单核1G内存20G系统盘，除了节点的地域其他的配置都是可以随时升级的，唯独地域是不可更换的，所以如果需要购买其他服务或者多台云主机，请事先考虑好地域选择，因为服务或主机之间不能跨地域交互。
3.  系统镜像我选择的是CentOs6.5的原生镜像，如果觉得配置环境比较困难或者嫌麻烦的话可以选择WordPress建站镜像，服务器则会自动搭建好WordPress的运行环境。 ![购买配置](http://www.scienjus.com/wp-content/uploads/2015/07/aliyun.png)
4.  付款成功后进入管理控制台，会发现已经有一台ECS主机正在启动中，新买的主机需要大概5分钟左右启动，然后就会变成运行中的状态。需要记住几个信息：服务器的外网ip地址，服务器的用户名(root)和服务器的密码(在购买时设置的密码)。

# 搭建环境

运行 WordPress 需要安装 Apache（或 Nginx）、PHP、MySQL 环境，流程如下：

1.  通过 SSH 远程连接云主机，这里我使用的是 putty（一个绿色小巧的远程连接工具），首先打开 putty ,在 Host Name 一栏填入服务器的外网 ip，Port 默认为 22 ，Connection type 默认为 SSH，之后点击 open，会弹出一个命令行窗口，如果连接上了会提示输入用户名和密码，将服务器的用户名和密码输入即可连接成功。 ![putty配置](http://www.scienjus.com/wp-content/uploads/2015/07/putty.png)
2.  在阿里云的 CentOS 系统下安装软件一般有3种选择：
    *   购买服务器时选择集成了运行环境的镜像（推荐完全不了解Linux系统操作的人选择）
    *   使用yum操作自动安装软件（推荐了解基本 CentOS 操作的人选择）
    *   自己下载源码，编译，安装，配置（推荐经验丰富者或大神选择）

    这里我选择的是第二种：使用yum操作自动安装软件，主要命令如下：
    
    1.  首先为了接下来的操作顺利，暂时关掉防火墙

        ```
        chkconfig iptables off
        ```

    2.  删除服务器中可能预先安装好的 Apache

        ```
        yum remove httpd
        ```

    3.  安装 Nginx

        ```
        yum -y install nginx
        ```

        如果没有写 `-y` 会出现提示选择 yes/no，-y 选择默认 yes
    4.  开启 Nginx 服务

        ```
        service nginx start
        ```

        如果顺利的话，此时打开浏览器输入服务器的外网 ip 会看见 Nginx 的默认页面 ![nginx](http://www.scienjus.com/wp-content/uploads/2015/07/nginx.png)
    5.  安装 PHP（这里仅安装了 php，php-fpm 和 php-mysql 足以支持 WordPress 的运行）

        ```
        yum -y install php php-fpm php-mysql
        ```

    6.  将 Nginx 映射到 PHP，这里假定 WordPress 的存放位置为 `/home/www/blog`

        ```
        vi /etc/nginx/nginx.conf
        ```

        将对应代码修改为

        ```
        location / {
          root   /home/www/blog;
          index  index.html index.htm index.php;
        }

        location ~ \.php$ {
          root                /home/www/blog;
          fastcgi_pass        127.0.0.1:9000;
          fastcgi_index       index.php;
          fastcgi_param       SCRIPT_FILENAME  /home/www/blog$fastcgi_script_name;
          include             fastcgi_params;
        }
        ```

    7.  修改 `php.ini` 配置

        ```
        vi /etc/php.ini
        ```

        增加一行配置

        ```
        cgi.fix_pathinfo = 1
        ```

    8.  启动 php-ftp 并重启 Nginx

        ```
        service php-fpm start
        service nginx restart
        ```

    9.  在网站根目录下创建一个测试文件 `vi /home/www/blog/index.php` 插入以下代码切换到浏览器刷新页面，首页如果显示php版本信息证明配置成功了 ![phpinfo](http://www.scienjus.com/wp-content/uploads/2015/07/phpinfo.png)
    10.  安装 MySQL

        ```
        yum -y install mysql mysql-server
        ```

    11.  启动服务

        ```
        service mysqld start
        ```

    12.  更改 root 账户密码

        ```
        /usr/bin/mysqladmin -u root password 'newPassword'
        ```

        提示输入密码时直接按回车
    13. 登录

        ```
        /usr/bin/mysql -u root -p
        输入密码
        ```

        如果能进入 MySQL 界面就说明已经安装好了

到此为止所有环境均已搭建完毕，接下来只需要安装 WordPress 即可完成建站了。

# 安装 WordPress

1.  首先到官网上获取最新版的 WordPress，例如我安装时最新版为 `4.2.2`，下载地址为[https://cn.wordpress.org/wordpress-4.2.2-zh_CN.zip](https://cn.wordpress.org/wordpress-4.2.2-zh_CN.zip)
2.  通过 SSH 登录到服务器，切换到网站根目录，下载这个文件

    ```
    cd /home/www/blog
    wget https://cn.wordpress.org/wordpress-4.2.2-zh_CN.zip
    ```

3.  解压，将所有文件都移动到根目录

    ```
    unzip -o wordpress-4.2.2-zh_CN.zip
    cd wordpress
    mv * ../
    ```

    如果有强迫症可以把压缩包和空文件夹也删了

    ```
    rm -rf wordpress-4.2.2-zh_CN.zip
    rm -rf wordpress
    ```

4.  进入 MySQL，创建一个用来存放 WordPress 数据的数据库

    ```
    /usr/bin/mysql -u root -p
    输入密码
    create database '数据库名'
    ```

5.  编辑 WordPress 配置文件，将数据库关联上

    ```
    mv /home/www/wp-config-sample.php /home/www/wp-config.php
    vi /home/www/wp-config.php
    ```

    修改以下属性

    ```
    define('DB_NAME', '数据库名');
    define('DB_USER', '用户名');
    define('DB_PASSWORD', '密码');
    define('DB_HOST', '主机地址(当前主机填localhost)');
    ```

6.  刷新浏览器，如果看到以下页面即可开始愉快的建站之旅 ![wordpress](http://www.scienjus.com/wp-content/uploads/2015/07/wordpress.png)

# 善后

1.  配置完网站后，记得将之前关闭的防火墙重新开启，并开启 80 端口的访问

    ```
    vi /etc/sysconfig/iptables
    ```

    在最底部追加代码

    ```
    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
    ```

    即可开启 80 端口，同时重新开启防火墙

    ```
    chkconfig iptables on
    ```

2.  将安装的程序注册为开机自动启动服务

    ```
    chkconfig mysqld on
    chkconfig php-fpm on
    chkconfig nginx on
    chkconfig vsftpd on
    ```

