# docker-lnmp

Linux + Nginx + MySQL + PHP

值得参考的官方 [Dockerfile](https://github.com/docker-library/docs)

之前还在想从网上下载的镜像能否逆编译出来它的 Dockerfile ，来学习一下它的创建过程，版本号，历史记录，基本镜像之类的，原来都在 Docker 的官方仓库中可以找到，如果想直接使用镜像反编译的话，暂时还不知道什么办法。

不过有官方源的官方镜像的 Dockerfile 已经很满足了，可以好好看一下。

## Linux

Debian 8 (jessie) 

并不是我想的，我其实比较喜欢在服务器上用 Cent OS ，但是 Docker 的官方仓库的官方镜像大多是采用 Debian。

## Nginx

用的是默认的 latest 版本，即 1.11.9 的版本。

```
[root@localhost php]# docker-compose exec nginx nginx -v
nginx version: nginx/1.11.9
```

工作目录在 `/usr/share/nginx/html` ,这是站的根目录。

主要的文件挂载有三处

- `./nginx/html:/usr/share/nginx/html` 将 `./nginx/html` 挂载到网站根目录上
- `./nginx/nginx.conf:/etc/nginx/nginx.conf` 将 `./nginx/nginx.conf` 设置为 Nginx 的配置文件
- `./nginx/ca:/etc/nginx/ca` 将 `./nginx/ca` 设置为 Nginx 的证书预留目录

一开始也想给日志做一个文件挂载出来，但是发现 Nginx 的日志默认输出到屏幕上，使用 `docker-compose logs nginx` 即可查看 Nginx 日志。

在 Nginx 的配置文件中设定用户为 `www-data`

开放端口 80 和 443，并映射到主机上

Nginx 并不需要直接连接 MySQL 数据库，只需要连接 PHP 即可。

因为在 Nginx 中的配置文件中需要填写 PHP 的地址，所以与 PHP 的连接中填上 PHP 的容器在 Nginx 的容器中的代称 PHP_ADDRESS ，这样就能在 Nginx 容器中使用 PHP_ADDRESS 来代替 PHP 的 IP 地址了。

查看容器的 IP 地址可以使用 `docker-compose exec nginx ip addr` 来查看。

## PHP

用的是默认的 latest 版本，即 7.1.1 

```
[root@localhost docker]# docker-compose exec php php -v
PHP 7.1.1 (cli) (built: Jan 24 2017 18:37:20) ( NTS )
Copyright (c) 1997-2017 The PHP Group
Zend Engine v3.1.0, Copyright (c) 1998-2017 Zend Technologies
```

本来也想只用原生的 php-fpm 就好了，但是在配置好之后发现，官方的镜像甚至连 MySQL 扩展都没有，简直不能忍。

查询了一下网上的 LNMP 的配置，PHP 的部分都是需要自己构造镜像，好吧，自己动手，丰衣足食。

写了一下 PHP 镜像构建的 Dockerfile ，主要就是安装了 mysqli, PDO 等几个与 MySQL 有关的扩展，发现在 PHP 7 上已经不再支持 mysql 扩展了，可能是安全性太低了吧。

如果使用 PHP 的过程中需要其他的扩展，需要 修改 Dockerfile 重新构建镜像，重新构建的命令为 `docker-compose build`

工作目录同样是在 `/usr/share/nginx/html` 根目录下。

文件挂载有三处

- `./nginx/html:/usr/share/nginx/html` 网站根目录必须挂载，否则会 404 
- `./php/php.ini:/etc/php.ini` 配置文件，未做改动。
- `./php/www.conf:/usr/local/etc/php-fpm.d/www.conf` 配置文件，修改用户和用户组为 www-data, www-data 与 Nginx 一致

PHP 需要连接 MySQL ，同时为 MySQL 设定一个代称，同时在环境变量中输入 MySQL 的 root 密码和默认使用 数据库。
  
## MySQL

用的是默认的 latest 版本，即 5.7.17 

```
[root@localhost docker]# docker-compose exec mysql mysql --version
mysql  Ver 14.14 Distrib 5.7.17, for Linux (x86_64) using  EditLine wrapper
```

文件挂载仅一处，为配置文件 `./mysql/my.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf`

在配置文件 `/etc/mysql/mysql.conf.d/mysqld.cnf` 已经设置好了字符编码为 utf8 。

```
mysql> show variables like "%character%";
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)

```

对外开放端口 3306 映射到主机上，这样在主机上可以通过 `mysql -u root -h 127.0.0.1 -p` 来登录

MySQL root 密码设为 `password` 

本来想将 MySQL 的数据目录 `/var/lib/mysql` 也挂载出来，但是因为权限问题作罢，下次直接自己写 Dockerfile 构建镜像吧。
