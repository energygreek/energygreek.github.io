---
date: 2023-07-08
tags: [wordpress]
---

# 安装wordpress记录
本文记录在debian上安装配置wordpress的补助，并配置了ftp功能，支持自更新wordpress。

## 安装wordpress
下载最新的包，解压到`/var/www/html`
## 安装配置数据库
安装mariadb, 创建数据库和用户
```
mysql -u root
CREATE DATABASE wordpressdb; 
CREATE USER wordpressuser@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpressdb.* TO wordpressuser@localhost;
```
## 安装配置nginx php
安装nginx和php-7.4，还有php一些组件
nginx 配置
```
server { 
  listen 80 default_server; 
  listen [::]:80 default_server;
  server_name your_domain.com;

  root /var/www/html;
  index index.php index.html index.htm;

  location / {
    # try_files $uri $uri/ =404;
    try_files $uri $uri/ /index.php?q=$uri&$args;
  }

  location ~ \.php$ {
      include snippets/fastcgi-php.conf;
      fastcgi_split_path_info ^(.+\.php)(/.+)$;
      fastcgi_pass unix:/run/php/php-fpm.sock;
  }
}
```

## 安装配置vsftpd
安装后，修改配置文件`/etc/vsftpd.conf`

```
local_enable=YES
write_enable=YES
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list
allow_writeable_chroot=YES
pm_service_name=ftp
```

### 添加用户，并限制权限

```
useradd [user_name]
passwd [user_name]
usermod -d /var/www/html/wordpress user_name -s /sbin/nologin

setfacl -m u:ftpsecure:r-x /var/www/html/

setfacl -R -m u:ftpsecure:rwx /var/www/html/wordpress
setfacl -R -d -m u:ftpsecure:rwx /var/www/html/wordpress
```
如果提示没有setfacl命令，则执行安装acl命令 `apt install acl`

### ftp遇到问题和解决办法

* 登陆提示"500 OOPS: could not read chroot() list file:/etc/vsftpd.chroot_list"，需要创建文件`/etc/vsftpd.chroot_list`

* 登陆ftp报530验证失败的问题，修改配置如下
```
pm_service_name=ftp
```
* 登陆提示"500 OOPS: vsftpd: refusing to run with writable root inside chroot()

```
allow_writeable_chroot=YES
```

* 登陆提示 500 OOPS: vsftpd: refusing to run with writable root ，意思是登陆者具备对目录的写权限，这是由于目录设定权限为777，所以要是重新设定为755
```
chmod 755 /srv/ftp
```

## 配置wordpress

修改`wordpress/wp-config.php` 文件, 添加如下内容

```
define('FTP_HOST', 'localhost');
define('FTP_USER', 'ftp');
define('FTP_PASS', '');

```
通过wordpress的api可以生成随机token，将输出替换到`wordpress/wp-config.php` 

```
$curl -s https://api.wordpress.org/secret-key/1.1/salt/

define('AUTH_KEY',         'uOeyPW$K[P}23x^0l##N+qP#xzUlIBV[ZQlATs!7J?+5^!w0*bgEw|V6)k:YU0en');
define('SECURE_AUTH_KEY',  'Ad*-`eK|U4Z*7g}7CdX<q0^EuGqT1Tt}CaDRgF%NX-|fmk(:BACtws+^_0Pb8ZRA');
define('LOGGED_IN_KEY',    ':Z$}=h~z-[]8W}wZ(|/;]ZY;U{>u3K>P|u6/d:6}&h*);0ewhrcGm8$/tPii#{,%');
define('NONCE_KEY',        'd}Ue8.+KPDGdH^%2t~fTljDj8e{q{raR!Q0piqM(gcQP&7-$L3u0u|!Bn(-gh/?B');
define('AUTH_SALT',        'UQh.k>4UBTK6IecZe~lL6|J/w^KFROIeCdZw^g=(x?(L+j(-|%C FCt =V*XVz+k');
define('SECURE_AUTH_SALT', 'hHr&dDCApb14yz@ks+;}mk,s<-[]KUj*UAhl0+<pqT*!j;wxc+[mU[czh =><#em');
define('LOGGED_IN_SALT',   'WO,U^A||&:E@2A2_|MS W1l`fguj-7E{Fz6QkC+OM96Ho49<agM?O=G2FloxuF?6');
define('NONCE_SALT',       'uDN.1::MJ>~9Yp+e39y%o?r >:+OcUSqg5_g<Jt:Sr!um?U:U|MJ2q2qKiLDfmr@');
```

更新数据库配置到wp-config.php

## 更新url
开始使用http访问，后面切换到https后一直提示错误说跳转太多次。后来发现原因是套了cloudflare，cloudflare设置的是前端https后端http, 后端以http访问真实服务器后又跳转到https。所以请求一直在cf<->wordpress之间无限跳转。所以首先要设置cloudflare的代理模式为严格安全，strict full safe，也就是前端和后端都用证书。然后修改wordpress的url方法有很多，之前通过直接修改数据库，这次发现wordpress有个wp的命令行工具，一键修改。

## vsftpd 配置默认目录
为local user配置默认目录，例如所有用户登录后进入/home目录, 添加配置`local_root=/home`，为匿名用户配置默认目录为tmp目录，则添加配置`anon_root=/tmp`

## 参考

https://serverfault.com/questions/544850/create-new-vsftpd-user-and-lock-to-specify-home-login-directory
https://shiftedbits.org/2012/01/29/enable-wordpress-automatic-updates-on-a-debian-server/
