---
layout: post
title:      "整理 Ubuntu 的各种小场景"
subtitle:   ""
date:       2019-03-02 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   Linux
---

## 序言

收集整理日常用到的各种小功能。

其实大部分功能在 [鸟哥私房菜](http://linux.vbird.org/linux_basic/) 都有详细的说明。

不过开的其它坑太多 (

有时间陆续补上。

## 系统启动时执行的服务

### 概述 1

-   目录 `/etc/init.d` 存放系统启动时执行的脚本。
-   目录 `/etc/rc?.d` 存放脚本在不同运行级别下的链接文件。
    其中链接命名格式为 `[S|K][num][shell_name]`，S 为启动，K 为关闭，num 为取值 0~99 的启动顺序，shell_name 即脚本名字。
-   使用 `update-rc.d` 对执行脚本进行管理。
-   Centos 使用 chkconfig 命令。

Ubuntu 系统运行级别：

| 级别 | 意义                 |
| ---- | -------------------- |
| 0    | 系统停机状态         |
| 1    | 单用户或系统维护状态 |
| 2~5  | 多用户状态           |
| 6    | 重新启动             |

### 命令实例

命令形式：

```bash
usage: update-rc.d [-n] [-f] <basename> remove
       update-rc.d [-n] <basename> disable|enable [ S|2|3|4|5]
        -n: not really
        -f: force
```

-   使用 `update-rc.d xxx defaults` 添加服务，相当于 `update-rc.d apachectl start 20 2 3 4 5 . stop 20 0 1 6 .`。
-   使用 `update-rc.d xxx defaults 20 80` 指定启动与关闭顺序。
-   使用 `update-rc.d -f xxx remove` 禁用服务。

### 自定义服务

在 `/etc/init.d` 新建服务脚本文件：

```bash
# !/bin/bash
### BEGIN INIT INFO
#
# Provides:  <可执行脚本名>
# Required-Start:   $local_fs  $remote_fs
# Required-Stop:    $local_fs  $remote_fs
# Default-Start:    2 3 4 5
# Default-Stop:     0 1 6
# Short-Description: initscript
# Description: This file should be used to construct scripts to be placed in /etc/init.d.
#
### END INIT INFO

## Fill in name of program here.
PROG="<可执行脚本名>"
PROG_PATH="<可执行脚本路径>" ## Not need, but sometimes helpful ( if $PROG resides in /opt for example ) .
PROG_ARGS=""
PID_PATH="/var/run/"

start ( ) {
    if [ -e "$PID_PATH/$PROG.pid" ]; then
        ## Program is running, exit with error.
        echo "Error! $PROG is currently running!" 1>&2
        exit 1
    else
        ## Change from /dev/null to something like /var/log/$PROG if you want to save output.
        $PROG_PATH/$PROG $PROG_ARGS 2>&1 >/var/log/$PROG &
    $pid=`ps ax | grep -i 'location_server' | sed 's/^\ ( [0-9]\{1,\}\ ) .*/\1/g' | head -n 1`

        echo "$PROG started"
        echo $pid > "$PID_PATH/$PROG.pid"
    fi
}

stop ( ) {
    echo "begin stop"
    if [ -e "$PID_PATH/$PROG.pid" ]; then
        ## Program is running, so stop it
    pid=`ps ax | grep -i 'location_server' | sed 's/^\ ( [0-9]\{1,\}\ ) .*/\1/g' | head -n 1`
    kill $pid

        rm -f  "$PID_PATH/$PROG.pid"
        echo "$PROG stopped"
    else
        ## Program is not running, exit with error.
        echo "Error! $PROG not started!" 1>&2
        exit 1
    fi
}

## Check to see if we are running as root first.
## Found at http://www.cyberciti.biz/tips/shell-root-user-check-script.html
if [ "$ ( id -u ) " != "0" ]; then
    echo "This script must be run as root" 1>&2
    exit 1
fi

case "$1" in
    start )
        start
        exit 0
    ;;
    stop )
        stop
        exit 0
    ;;
    reload|restart|force-reload )
        stop
        start
        exit 0
    ;;
    ** )
        echo "Usage: $0 {start|stop|reload}" 1>&2
        exit 1
    ;;
esac
```

使用 `update-rc.d` 进行设置即可工作。

另外，使用工具 `sysv-rc-conf` 可进行更直观的管理：

`sudo apt-get install sysv-rc-conf`

运行 `sudo sysv-rc-conf` 即可打开。

## 挂载磁盘

### 概述 2

磁盘被手动挂载后，重启系统后仍需要手动挂载。

而将挂载信息写入 `/etc/fstab` 文件中即可使其在系统启动时挂载。

### 配置说明

配置项有 6 列参数，

| 列数 | 意义             | 说明                                    |
| ---- | ---------------- | --------------------------------------- |
| 1    | 设备名           | ---                                     |
| 2    | 挂载点           | ---                                     |
| 3    | 磁盘格式         | 包括 ext2、ext3、reiserfs、nfs、vfat 等  |
| 4    | 文件系统参数     | 常用 defaults，其包括多个默认参数       |
| 5    | 是否被 dump 备份 | 0 否，1 每日，2 不定日期               |
| 6    | 是否校验扇区     | 0 否，1 最早即根目录，2 等 1 完成后校验 |

我在 Virtualbox 虚拟机上挂载共享文件夹时的配置：

`<共享文件夹名称> /mnt/share/ vboxsf defaults 0 0`

注意需要在 Virtualbox 中关闭共享文件夹的自动挂载选项。

## 定时任务

### 概述 3

`/etc/crontab` 一般为系统管理员制定的 crontab。

额外配置项有：

```bash
SHELL=/bin/bash                    ## 指定命令执行的环境
PATH=/sbin:/bin:/usr/sbin:/usr/bin ## 执行命令路径
MAILTO=root                        ## 执行信息包括 echo 会通过邮件发送 root 用户
```

`/var/spool/cron` 存放每个用户包括 root 的 crontab 任务。

任务以用户名命名，每个用户最多有一个 crontab 文件。

通过 `crontab -e` 命令创建或编辑。

crontab 权限由 `/etc/cron.allow` 和 `/etc/cron.deny` 控制。

| cron.allow | cron.deny | 拥有权限的用户                                |
| ---------- | --------- | --------------------------------------------- |
| ×          | ×         | 所有用户                                      |
| ×          | √         | 非 deny 的用户                                |
| √          | ×         | alolow 的用户                                 |
| √          | √         | allow 或非 deny 的用户，若均有则以 allow 为准 |

### 配置实例

```bash
### 命令格式:
# min    hour     day   month weekday ( 0==7 )
# ( 0~59 ) ( 0~23 ) ( 1~31 ) ( 1~12 ) ( 0~7 ) [user ] command

3,15 8-11 */2 * * command              ## 每隔两天的上午 8 点到 11 点的第 3 和第 15 分钟执行
0 23-7/2,8 * * * command              ## 晚上 11 点到早上 8 点之间每两个小时和早上八点
0 1 * * * * run-parts /etc/cron.hourly ## 每小时执行目录内的脚本
```

## 回收站

安装 `trash-cli`，命令概述：

| 命令          | 功能                   |
| ------------- | ---------------------- |
| trash-put     | 移入回收站             |
| trash-rm      | 删除回收站中的单个文件 |
| trash-list    | 查看回收站中的文件     |
| trash-empty   | 清空回收站             |
| restore-trash | 还原文件               |

回收站路径为 `~/.local/share/Trash/`

可用它替代 `rm` 命令，在 `~/.bashrc` 中写入：

`alias rm='trush-put'`

## vim 配置

### 概述 4

参考自阮一峰老师的博文 [Vim 配置入门](http://www.ruanyifeng.com/blog/2018/09/vimrc.html)

全局配置位于 `/etc/vim/vimrc`，个人配置位于 `~/.vimrc`

关闭选项一般在打开选项前加上 no ：`set nonumber`

查询配置项是否开启：`: set number?`

查看配置项帮助：`: help number`

### 配置项

```vim
"" 基本配置
set nocompatible               "" 不与 vi 兼容
syntax on                      "" 语法高亮
set showmode                   "" 底部显示当前模式
set showcmd                    "" 显示当前键入的指令
set mouse=a                    "" 支持鼠标
set encoding=utf-8
set t_Co=256                   "" 启动 256 色
filetype indent on             "" 自动检查文件类型对应的缩进规则，缩进规则位于~/.vim/indent/xxx.vim

"" 缩进
set autoindent                 "" 回车换行保持缩进，但会影响粘贴文本
set tabstop=4                  "" Vim 显示的 Tab 空格数
set shiftwidth=4               "" >>、<<、== 的字符数
set expandtab                  "" 将 Tab 转为空格
set softtabstop=4              "" Tab 的空格数

"" 外观
set number                     "" 行号
"set relativenumber             "" 其它行相对光标航行的相对行号
set cursorline                 "" 光标所在行高亮
set textwidth=80               "" 行宽
"set wrap                       "" 自动折行
set linebreak                  "" 只有遇到特定符号而非单词内部才会折行
set wrapmargin=2               "" 折行与编辑窗口的距离
set scrolloff=5                "" 垂直滚动时，光标与顶部/底部的位置
set sidescrolloff=15           "" 水平滚动时，光标与行首/行尾的位置
set laststatus=2               "" 显示状态栏，0 否，1 只在多窗口时，2 是
set ruler                      "" 状态栏显示光标当前位置

"" 搜索
set showmatch                  "" 光标在括号时，高亮另一个括号
set hlsearch                   "" 高亮匹配结果
set incsearch                  "" 搜索时每输入一个字符，就自动跳到第一个匹配
set ignorecase                 "" 忽略大小写
"set smartcase                  "" 如果打开了 ignorecase，则只对首字母大小写敏感

"" 编辑
"set spell spelllang=en_us      "" 英语单词拼写检查
set nobackup                   "" 不创建备份，文件保存会额外创建结尾带 ~ 的备份文件
set noswapfile                 "" 不创建交换文件，用于系统崩溃时恢复，开头带 . 结尾带 .swp
set noundofile                 "" 不保留撤销记录，记录文件开头带 .un~
"set backupdir=~/.vim/.backup/  "" 备份文件保存位置
"set directory=~/.vim/.swp/     "" 交换文件保存位置
"set undodir=~/.vim/.undo/      "" 撤销记录文件保存位置
set autochdir                  "" 自动切换工作目录，主要用于多文件情况
set errorbells                 "" 出错发出响声
set novisualbell               "" 出错时不发出视觉提示
set history=1000               "" 记住的历史操作
set autoread                   "" 文件监视，若编辑时文件外部改变则提示
"set listchars=tab:»■,trail:■
"set list                       "" 行尾多余空格或 Tab 可见
set wildmenu
set wildmode=longest: list,full "" 命令模式操作指令按 Tab 补全
```

当粘贴文本时出现缩进混乱问题，则应该通过 `set paste` 设置粘贴模式，再进行粘贴

## 日志文件管理

### 概述 5

`/etc/logrotate.conf` 默认配置文件

`/etc/logrotate.d/*` 具体程序的配置文件

`logrotate [ OPTION...] <configfile>` 参数：

| 参数 | 功能                                  |
| ---- | ------------------------------------- |
| -d   | 测试配置文件是否错误                  |
| -f   | 强制转储                              |
| -m   | --mail = command，日志发送到邮箱        |
| -s   | --state = statefile，使用制度的状态文件 |
| -v   | 显示转储过程                          |

`/var/lib/logrotate/status` 记录了具体执行情况

### 配置项说明

以 Nginx 日志配置为例：

```bash
/var/log/nginx/*.log {
    daily
    rotate 7              ## 日志转储数
    size 5k               ## 日志文件最大大小，超过则转储
    compress              ## 通过 gzip 压缩转储日志
    delaycompress         ## 转储日志在下一次转储才压缩
    create 0640 root root ## 转储创建新文件属性
    copytruncate          ## 若日志在打开，则备份当前日志并截断
    missingok             ## 丢失日志不报错，继续轮转
    notifempty            ## 日志为空则不轮转
    noolddir              ## 转储后的与当前的日志于同一目录下
    dateext
    dateformat -%Y-%m-%d  ## 日志名插入日期值

    sharedscripts         ## 对一条脚本，所有日志都轮转后统一执行一次，否则每次轮转都执行一次
    prerotate             ## 转储前执行的脚本
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate            ## 转储后执行的脚本
        invoke-rc.d nginx rotate >/dev/null 2>&1
    endscript
    postrotate
        test -r /var/run/nginx.pid && kill -USR1`cat /var/run/nginx.pid`
    endscript
}
```
