---
layout: post
title:      "从零开始在 Ubuntu 16.04.5 搭建 Ghost 博客"
subtitle:   ""
date:       2019-01-25 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   安装/部署
---

## 序言

安装过程其实跟 [Ghost 文档](https://docs.ghost.org/install) 一致

不过我需要在本地 VirtualBox 虚拟机上也安装博客，这样可以在本地修改主题后同步至服务器上

另外还为服务器配置了 SSL 证书

因此写下该博文，记录过程中遇到的问题与所采用的解决方案

## 准备工作

一个 Ubuntu 16.04.5 服务器，一个解析、备案均完成的域名

版本相关信息如下：

-   Ubuntu 16.04.5
-   Nginx 1.10.3
-   PHP 7.2
-   MySQL 5.7.24
-   Node.js 10.13.0 ( 该版本 Ghost 可支持的最高版本，可用 nvm 降低版本，详情见下文 )
-   Ghost 2.11.1
-   Ghost-CLI 1.9.9

## 安装 Nginx + PHP + MySQL + phpMyAdmin

其实搭建该博客无需用到 PHP 与 phpMyAdmin，但是日后其它事情会用到，顺便附上

### 安装

```bash
sudo apt-get update
sudo apt -y install software-properties-common
sudo add-apt-repository ppa: ondrej/php
sudo apt update
sudo apt install php7.2-fpm php7.2-mysql php7.2-curl php7.2-gd php7.2-mbstring php7.2-xml php7.2-xmlrpc php7.2-zip php7.2-opcache --fix-missing -y
sudo sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/' /etc/php/7.2/fpm/php.ini
sudo apt-get install php-gettext

# 安装 MysQL 过程会要求配置数据库用户密码
# 另外过程中某处会提示选择 apache/lighthttp，选择 lighthttp
sudo apt-get install mysql-server mysql-client
sudo apt-get install phpmyadmin
```

此时在浏览器输入服务器对应 IP 或域名，若能看到 Nginx 的默认首页，则安装 Nginx 成功

### 配置 php-fpm

打开配置文件：

```bash
vim /etc/php/7.2/fpm/pool.d/www.conf
```

修改 listen 处为：

```nginx
; The address on which to accept FastCGI requests.
; Valid syntaxes are:
;   'ip.add.re.ss: port'    - to listen on a TCP socket to a specific IPv4 address on
;                            a specific port;
;   '[ip:6: addr: ess]: port' - to listen on a TCP socket to a specific IPv6 address on
;                            a specific port;
;   'port'                 - to listen on a TCP socket to all addresses
; ( IPv6 and IPv4-mapped ) on a specific port;
;   '/path/to/unix/socket' - to listen on a unix socket.
; Note: This value is mandatory.
;listen = /run/php/php7.2-fpm.sock
listen = 9000
```

重启 php-fpm：

```bash
service php7.2-fpm restart
```

### php-fpm 配置说明
php-fpm 可使用 UNIX Socket 和 TCP 两种监听方式

关于两种方式的区别，引用一段我觉得比较好的解释：

>It's basically a tradeoff between performance and flexibility. Unix domain sockets will give you a bit better performance, while a socket connected to localhost gives you a bit better portability. You can easily move the server app to another OS by just changing the IP address from localhost to a different hostname.
>
>A Unix domain socket uses the local file system to create an IPC mechanism between the server and client processes. You will see a file in /var somewhere when the Unix domain socket is connected.
>
>If you are looking for purely the ultimate performance solution you may want to explore a shared memory IPC. But, that's a little bit more complex.

简单地说就是 UNIX Socket 方式性能更好但不易分离和移植

TCP 方式则更容易进行分离如数据库和 Web 服务器分开的场景

不过嘛只有到服务器负载较高的时候才有较大的性能差异，个人服务器就怎么方便怎么来吧

直接监听 9000 端口即好

## 安装 Node.js + Ghost + SSL

**注意：该大步骤中应使用一个非 root，且处于 sudo 用户组的用户执行命令**

添加用户至 sudo 组的命令如下：

```bash
sudo usermod -aG sudo <user>
su <user>
```

### 安装 Node.js

按照 Ghost 文档的安装方法为：

```bash
# Add the NodeSource APT repository for Node 8
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash

# Install Node.js
sudo apt-get install -y nodejs
```

但若通过常规方法安装，Node.js 版本可能会过高：

```bash
sudo apt-get install nodejs
sudo apt install nodejs-legacy
sudo apt install npm
```

这时可以使用 nvm，管理 Node.js 版本：

```bash
# 下面命令使用一个非 root 用户运行
cd ~
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.31.2/install.sh | bash
```

安装完后可能需要重启服务器，可用下面命令验证安装成功：

```bash
command -v nvm
# 输出 "nvm"，即安装成功
```

本人搭建 Ghost 时对应可用的最新 Node.js 版本为 10.13.0，所以执行下面命令：

```bash
nvm install 10.13.0
nvm use 10.13.0

# 可检验 node 版本
node -v
```

若不知道当前 Ghost 版本对应的最新 Node.js 版本

可以继续进行下部分安装 Ghost 的步骤，直至 Ghost 运行失败时，运行命令`ghost doctor`

即可知道其支持的最新 Node.js 版本

 ( 注：nvm 更改版本后只对安装 nvm 的用户有效 )

### 安装 Ghost

执行一系列命令：

```bash
sudo npm install ghost-cli@latest -g

# We'll name ours 'ghost' in this example; you can use whatever you want
sudo mkdir -p /var/www/ghost

# Replace <user> with the name of your user who will own this directory
sudo chown <user>:<user> /var/www/ghost

# Set the correct permissions
sudo chmod 775 /var/www/ghost

# Then navigate into it
cd /var/www/ghost

ghost install
```

`ghost install`过程中一系列的问题参考 [Ghost 文档此处](https://docs.ghost.org/install/ubuntu/#install-questions)，其中 SSL 可先不配置

### 安装  SSL

SSL 证书使用 [Let's Encrypt](https://letsencrypt.org/)，[安装文档](https://certbot.eff.org/lets-encrypt/ubuntutrusty-nginx)

不过直接输入下面命令安装就好：

```bash
sudo apt-get install letsencrypt
```

然后停止 Nginx 以释放端口，获取证书

```bash
sudo service nginx stop
sudo letsencrypt certonly --standalone
```

过程中需要输入邮箱地址，然后输入域名

若希望访问博客时中不带 www，以使 URL 更简洁，则输入`example.com`格式的域名即可

若操作成功后，存在目录`/etc/letsencrypt/live/example.com/`，则表示注册证书成功

证书有效期为 90 天，证书过期后

同样停止 Nginx，运行`sudo letsencrypt certonly --standalone`即可更新证书

## 配置 Nginx

### 站点配置说明

来到目录`/etc/nginx`下，站点的配置相关文件与文件夹如下：

```bash
├── conf.d
├── nginx.conf
├── sites-available
├── sites-enabled
```

关于站点的设置，有三种配置方式：

1.  直接写入`nginx.conf`中
2.  在`conf.d`中新建站点文件
3.  在`sites-available`中新建站点文件，然后在`sites-enabled`中创建软链接

其实三种方案原理是一样的，见`nginx.conf`，有`include`项：

```bash
# #
# Virtual Host Configs
# #

include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

但直接写入`nginx.conf`中不便于管理

用`conf.d`管理站点比较方便，但若需要临时关闭几个站点而日后又可能重启站点时，则需要将配置文件移出或重命名

而`sites-available`中可存放所有站点的配置项，然后将实际需要使用的站点软链接到`sites-enabled`即可

本人选用`sites-*`方案进行站点的配置：

### 配置过程

首先上面安装过程可能自动为`nginx.conf`添加了配置内容

所以先打开`nginx.conf`，删掉`http`块段里存在的`server`块段

然后移除在`sites-available`下的默认站点文件`default`，预防万一也可以备份一下

然后新建两个分别监听 80、443 端口的配置文件

**注意，同一个端口的配置好像只能写在同一个文件内，否则可能出错
如果读者有较好的配置方案的话，希望能和博主分享下~**

为了更直观，分别命名为`80`、`443`

然后创建软链接到`sites-enabled`：

```bash
ln -s /etc/nginx/sites-available/80 /etc/nginx/sites-enabled/80
ln -s /etc/nginx/sites-available/443 /etc/nginx/sites-enabled/443
```

目录结构如下：

```bash
├── sites-available
│   ├── 80
│   └── 443
├── sites-enabled
    ├── 80 -> /etc/nginx/sites-available/80
    └── 443 -> /etc/nginx/sites-available/443
```

然后配置文件`443`：
```nginx
server {
        index index.html index.htm index.nginx-debian.html;
### server_name example.com www.example.com;
        location / {
                proxy_set_header   X-Real-IP $remote_addr;
                proxy_set_header   Host      $http_host;
                proxy_pass         http://127.0.0.1:2368;
        }
### access_log /var/log/nginx/example-access.log;
### error_log  /var/log/nginx/example-error.log error;

        listen 443 ssl;
        listen [:: ]:443 ssl;
### ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
### ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
### server_name mysql.example.com;
        root /usr/share/phpmyadmin;

        index index.html index.htm index.php;

        charset utf-8;

        location / {
                try_files $uri $uri/ /index.php?$query_string;
        }

        access_log /var/log/nginx/phpmyadmin-access.log;
        error_log  /var/log/nginx/phpmyadmin-error.log error;

        sendfile off;

        client_max_body_size 100m;

        include fastcgi.conf;

        location ~ /\.ht {
                deny all;
        }

        location ~ \.php$ {
                fastcgi_pass   127.0.0.1:9000;
                #fastcgi_pass /run/php/php7.2-fpm.sock;
                fastcgi_index  index.php;
                fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                include  fastcgi_params;
        }

        listen 443 ssl;
        listen [:: ]:443 ssl;
### ssl_certificate /etc/letsencrypt/live/mysql.example.com/fullchain.pem;
### ssl_certificate_key /etc/letsencrypt/live/mysql.example.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}
```

再配置文件`80`：
```nginx
server {
        listen 80;
        listen [:: ]:80;
### server_name example.com www.example.com;
        return 301 https://$server_name$request_uri;
}

server {
        listen 80;
        listen [:: ]:80;
### server_name mysql.example.com;
        return 301 https://$server_name$request_uri;
}
```

注意需要将带`###`前缀的行处里的域名修改成自己对应的域名，并去掉`###`

### 站点配置讲解

#### 443

443 端口是 https 对应的端口，当带有`https://`前缀访问 URL 时，便会通过 443 端口访问

第一个`server`块段将请求转发到了`http://127.0.0.1:2368`，该地址是 Ghost 默认的访问 URL

所以访问`https://example.com`或`https://www.example.com`后，即可访问 Ghost 博客

因为证书注册为`https://example.com`形式

因此访问`https://www.example.com`时会自动转为`https://example.com`

第二个`server`块段为 phpMyAdmin 的配置，访问`https://msyql.example.com`时即可访问该站点

注意该站点需要解析 PHP 文件，因此需要加上配置项`location ~ \.php$`

其将会请求交给 FastCGI 接口，配置项写上对应的 php-fpm 的监听方式

本人选用 TCP 监听方式，所以配置项为`127.0.0.1:9000`

#### 80

该配置文件用于将 http 访问重定向为 https 访问，实现强制 https 访问站点

原理简单，只需返回 301 重定向即可

### 验证配置

重启 Nginx：

```bash
service nginx restart
```

访问`https://example.com`，即可浏览博客首页

访问`https://example.com/ghost`，可进入博客后台

访问`http://www.example.com`，可见链接转为了`https://example.com`

## 本地 VirtualBox 下搭建 Ghost 博客

### 目的

Ghost 博客原理为提供了一系列接口，根据这些接口，即可开发不同的主题

Ghost 安装好后，默认自带了主题`casper`，其对应路径为`/var/www/ghost/content/themes/capser`

当想更换主题时，可找到想更换的主题对应的 git 仓库地址

将其 clone 到`/var/www/ghost/content/themes`下面，然后重启 Ghost

此时再打开 Ghost 后台 -> Design，即可更换主题

当想要修改主题时，即可修改主题内部的代码，或开发新功能

为了使开发过程更规范可控，应在本地进行开发，开发完成后，服务器与本地同步主题代码

因为我的本机是 Windows 系统，因此要借助 VirtualBox 搭建博客，于是多了很多配置上的问题

### VirtualBox 配置

准备好一个安装好的 Ubuntu Server 16.04.5 版本的 VirtualBox 虚拟机

#### 桥接网卡配置

非分配固定 ip 设置的网络可以使用桥接网卡

虚拟机网卡设置为桥接模式，注意网卡选择为本机上网所用的网卡

然后打开网卡配置文件：

```bash
sudo vim /etc/network/interfaces
```

里面的`lo`为本地回环，不用修改

而`enp0s*`为不同网卡的对应配置，找到对应桥接网卡的`enp0s*`

 ( 本人`enp0s3`对应网卡 1 配置，`enp0s8`对应网卡 2 配置，不知道不同机器是否一样，仅供参考 )

根据主机上对应网卡的配置 ( 在命令行输入`ipconfig /all`查看 )，配置对应桥接网卡的静态 IP

其中 IP 地址中第 4 个数字部分可任取与主机不同的值，网关、子网掩码则与主机一致

例子：

```bash
auto enp0s8
iface enp0s8 inet static
address 192.168.2.156
netmask 255.255.255.0
gateway 192.168.2.1
```

重启网络：

```bash
sudo /etc/init.d/networking restart

# 可查看网卡连接情况
ifconfig
```

配置成功后，虚拟机与主机可以通过子网 IP 地址互相 ping 通

虚拟机也可以 ping 通外网

#### Host-Only 网络配置

分配固定 ip 的网络 ( 如 Dr.com 校园网 ) 可使用该方式

首先在 VirtuaBbox 的「管理」→「主机网络管理器」创建一个新的 Host-Only 适配器

然后设置为手动配置网卡

IPv4 地址设置为 192.168.x.1, x 为任意合法值

网络掩码为 255.255.255.0

然后同桥接网卡设置，地址填为 192.168.x.y，无需填写网关

配置好重启，主机与虚拟机即可互 ping

需注意该方式下要连上外网还额外需要一个 NAT 模式的网卡

在 VirtualBox 设置好后，配置 interfaces 如下：

```bash
auth enp0s3
iface enp0s3 inet dhcp
```

#### 安装本地模式 Ghost

Ghost 有提供本地安装模式，本地安装模式下数据库由 SQLite3 实现

因此本地安装只需要安装 Nginx 与 Node.js 即可

然后 Ghost 安装命令换为`ghost install local`，其它步骤一致

#### 配置共享文件夹

首先安装增强功能：

```bash
sudo apt-get install dkms
sudo apt-get install build-essential
```

配置好后关闭虚拟机，配置共享文件夹，记住共享文件夹的名称 ( 不带 sf\_ 前缀 )

并将共享文件夹的自动挂载选项关闭

然后在本机管理员权限下的 cmd 执行以下命令：

`"VirtualBox\VBoxManage.exe 的路径" setextradata <虚拟机名字> VBoxInternal2/SharedFoldersEnableSymlinksCreate/<共享文件夹的名称> 1`

然后启动虚拟机，运行挂载文件夹命令：

```bash
sudo mkdir /mnt/share
sudo mount -t vboxsf <共享文件夹的名称> /mnt/share
# /mnt/share 为共享文件夹在虚拟机内的路径，可以随意设置
```

此时共享文件夹可正常工作

但手动挂载方式每次开机都要执行一次

因此应编辑系统的挂载配置文件，使其每次开机启动后自动挂载

编辑文件`/etc/fstab`，在最后加上一行：

```bash
<共享文件夹的名称> /mnt/share/ vboxsf defaults 0 0
```

下次开机即可生效

配置好共享文件夹后，即可 clone 主题项目到上面

然后创建软链接到`/var/www/ghost/content/themes`下面，重启 Ghost

即可使用并在本地修改主题，修改好后更新到服务器上，即可实现同步修改

## 其它问题

### Public API 报错

Public API 是 Ghost 提供的一系列 API

但其在 Ghost 2.11.0 已经弃用并被 Content API 取代

Ghost 管理后台 -> Labs 内已经没有开启它的选项

所以依赖它开发的主题均会出现报错，不过也是有解决方案的

在主题的`package.json`内找到`engines`项，修改为：

```json
"engines": {
    "ghost": ">=2.0.0",
    "node": ">= 6",
    "ghost-api": "v2"
},
```

即可使其正常工作

### 更换启动 Ghost 的用户

Ghost 的启动权限不仅受 Linux 权限系统控制，还与用户根目录内的`.ghost`配置文件有关

因此要更换启动 Ghost 的用户时，不仅要更改文件夹权限，还要复制配置文件

设新旧用户分别为 new_user、old_user，执行命令：

```bash
sudo cp /home/<old_user>/.ghost/config /home/<new_user>/.ghost/config

cd /var/www
sudo find . -group <old_user> -user <old_user> -exec chown <new_user>:<new_user> {} \;
```

注意若使用 nvm 控制版本，则新用户也需要安装 nvm 并配置
