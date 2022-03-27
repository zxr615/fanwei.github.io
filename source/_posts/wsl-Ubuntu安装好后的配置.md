---
title: wsl-Ubuntu安装好后的配置
categories: 运维
tags:
  - wsl2
  - Linux
  - Ubuntu
date: 2021-05-24 23:20:45
---

## wsl 官方文档

[https://docs.microsoft.com/zh-cn/windows/wsl/](https://docs.microsoft.com/zh-cn/windows/wsl/)



## Ubuntu server 

### 链接 wifi

[https://linux.cn/article-12628-1.html](https://linux.cn/article-12628-1.html)

查看网卡名称

```shell
ls /sys/class/net
```

修改配置

```shell
vim /etc/netplan/xxx-wifi.yaml # 
```

```yaml
network:
  version: 2
  wifis:
    wlan0:
        dhcp4: true
        optional: true
        access-points:
            "网络名称":
                password: "wifi密码"
```

应用

```shell
sudo netplan apply
```

注：生成配置，似乎不执行也行

```shell
sudo netplan generate
```



## 切换 apt 源

[https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)

```console
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak && sudo vim /etc/apt/sources.list
```

```console
sudo apt update
```

### 问题

- `apt update` 一直提示 443 

  安装系统的时候有一项 `proxy` 误设置成了软件源链接，导致代理出错，将系统代理去掉

  系统代理位置

  ```shell
  /etc/apt/apt.conf.d/90curtin-aptproxy
  ```

  ```
  /etc/systemd/system/snapd.service.d/snap_proxy.conf
  ```

  

## 配置 SSH

### 生成秘钥

```shell
ssh-keygen -t rsa -C "zxr615@foxmail.com"
```

> 创建 `RSA` `DSA` `ECDSA`  `ED25519` 到 `/etc/ssh/*`

```shell
ssh-keygen -A
```

### 配置文件

#### 服务端

```shell
sudo vim /etc/ssh/sshd_config
```

```shell
# 允许公钥登录
PubkeyAuthentication yes
# 公钥文件
AuthorizedKeysFile .ssh/authorized_keys
# 允许密码登录
PasswordAuthentication yes
```

#### 客户端

查看公钥

```
cat ~/.ssh/id_rsa.pub
```

客户端中的「id_rsa.pub」内容追加进服务端的 「authorized_keys」中

```shell
sudo vim ~/.ssh/authorized_keys
```

免密登录

```shell
SSH fanwei@192.168.0.1
```

### 别名登录,区分多个服务器

客户端：

```shell
vim ~/.ssh/config
```

写入

```
Host ubuntu
    HostName fanwei.cn
    Port 22
    User fanwei
    IdentityFile ~/.ssh/id_rsa
    
Host ubuntu2
    HostName fanwei2.cn
    Port 22
    User fanwei
    IdentityFile ~/.ssh/id_rsa
```

登录

```shell
ssh ubuntu
```

```shell
ssh ubuntu2
```

### 重启生效

```
sudo service ssh restart
```



## sudo免输入密码

```shell
visudo
```


fanwei 用户执行`任何命令`都不需要输入密码

```shell
fanwei ALL=(ALL) NOPASSWD: ALL
```

fanwei 用户执行 `vim`命令时不需要输入密码

```shell
fanwei ALL=(ALL) NOPASSWD: vim
```



## service命令

### 目录

```shell
cd /etc/init.d
```



## 终端代理

```shell
echo >> ~/.zshrc "export ALL_PROXY=socks5://172.24.208.1:7890"
```

```
source ~/.zshrc
```





## 服务自启

### 定义自启服务

```shell
sudo vim /etc/init.wsl
```

### 写入自启内容

```bash
#! /bin/sh
/etc/init.d/ssh $1
/etc/init.d/mysql $1
```

### 修改权限

```shell
sudo chmod +x /etc/init.wsl
```

### 启动服务

```shell
sudo /etc/init.wsl [start|stop|restart]
```

### Windows 端

`Win` + `R` 输入 `shell:startup` 

新建文件：`wslStartUp`

```vbscript
Set ws = CreateObject("Wscript.Shell")
ws.run "wsl -d Ubuntu -u root /etc/init.wsl start", vbhide
```

> 如果遇到启动失败，把 `wsl -d Ubuntu -u root /etc/init.wsl start` 在命令行中运行即可看到错误



## 安装 net-tools

```shell
sudo apt install net-tools
```

```shell
ifconfig
```



## 安装gcc

```shell
sudo apt install gcc
```



## 安装 git

### 安装

```console
sudo apt install git
```

### 配置

设置名称&邮箱

```console
git config --global user.name "fanwei"
```

```console
git config --global user.email "zxr615@foxmail.com"
```

不转义(git status 正常显示中文)

```shell
git config --global core.quotepath false
```



### 安装 zsh

### zsh

```shell
sudo apt install zsh
```

### oh-my-zsh

```shell
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

```console
Do you want to change your default shell to zsh? [Y/n] Y
你想把你的默认shell改为zsh吗？[Y/n] Y
```

### 插件

zsh-autosuggestion 自动补全

```shell
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

zsh-syntax-highlighting  命令高亮

```shell
git clone git://github.com/zsh-users/zsh-syntax-highlighting $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```

### 配置

```shell
vim ~/.zshrc
```

```shell
plugins=(
  git
  zsh-autosuggestions
  # 命令是否正确提示，放在最下面
  zsh-syntax-highlighting
)
```



## php&php-fpm&composer

### php

```shell
sudo apt install php
```

ext-curl

```shell
sudo apt install ext-curl
```

ext-dom

```
sudo apt install php7.4-xml
```

> 指定php版本，不指定版本使用默认版本

### php-fpm

```shell
sudo apt install php-fpm
```

###  composer

```shell
sudo apt install composer
```

更改源

```sehll
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```

多线程

```shell
composer global require hirak/prestissimo -vvv
```



### 启动

```
service php7.4-fpm start
```

### 启动脚本软链

> 启动时可用 `service php-fpm start`

```
sudo ln -s /etc/init.d/php7.4-fpm /etc/init.d/php-fpm
```



## 安装 go

[https://golang.org/doc/install?download=go1.16.4.linux-amd64.tar.gz](https://golang.org/doc/install?download=go1.16.4.linux-amd64.tar.gz)



## 安装 nodejs

### 官方文档

[https://github.com/nodesource/distributions#installation-instructions](https://github.com/nodesource/distributions#installation-instructions)

```shell
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
```

```shell
sudo apt install -y nodejs
```

```shell
node -v
```



## 安装 nginx

```shell
sudo apt install nginx
```



## 安装 redis

### 安装

```shell
sudo apt install redis
```

### 启动

```shell
service redis-server start
```



## 安装 mysql

```console
sudo apt install mysql-server
```

### 启动

```shell
sudo service mysql start
```

```shell
sudo service mysql stop
```

```shell
sudo service mysql restart
```

### 设置安全向导

```console
sudo mysql_secure_installation
```

```console
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) :N
您是否要继续使用提供的密码？（按y | Y表示是，按其他任意键表示否）：N

Please set the password for root here.
请在这里为root设置密码。

Remove anonymous users? (Press y|Y for Yes, any other key for No) :y
删除匿名用户？Y

Disallow root login remotely? (Press y|Y for Yes, any other key for No) :N
不允许远程登录根目录？n

Remove test database and access to it? y
删除测试数据库和对它的访问？ y

Reload privilege tables now? y
现在重新加载权限表？y
```

### 创建一个mysql用户

```shell
create user fanwei identified by '123456';
```

### 查看用户权限

```shell
 SHOW GRANTS FOR fanwei;
```

```console
+------------------------------------+
| Grants for fanwei@%                |
+------------------------------------+
| GRANT USAGE ON *.* TO `fanwei`@`%` |
+------------------------------------+
```

### 给予root权限

```
# 分配 所有权限 所有数据库.所有表 给 fanwei用户
GRANT ALL ON *.* TO fanwei;
```

### 刷新用户权限

```shell
FLUSH PRIVILEGES;
```

### 查看用户

```shell
SELECT * FROM `user` WHERE User="fanwei"\G
```

### 配置文件

```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

监听所有 ip

```
bind-address            = 0.0.0.0
mysqlx-bind-address     = 0.0.0.0
```

### 问题

#### 查询配置信息

```shell
mysqld --verbose --help | more
```

```
...
Default options are read from the following files in the given order:
默认选项按照给定的顺序从以下文件中读取
/etc/my.cnf /etc/mysql/my.cnf /opt/homebrew/etc/my.cnf ~/.my.cnf
The following groups are read: mysqld server mysqld-8.0
The following options may be given as the first argument:
...
```

### 免密登录

常常会遇到密码忘记，或者刚设置好然后登录不上的情况，这个时候就需要免密进入后再进行配置

```shell
vim ~/.my.cnf
```

加入配置，重启

```she
[mysqld]
# 免密登录
skip-grant-tables
```

重新进入

```shell
mysql -u root -p
```

### 常用配置

> [mysqld] 下设置

#### 开启查询日志

```shell
# 开启查询日志
general-log=1
# 设置日志位置
general-log-file=
```

或者命令行中执行

```
set GLOBAL general_log='ON';
```

查询是否开启

```shell
SHOW VARIABLES LIKE 'general%';
```

```console
mysql> SHOW VARIABLES LIKE 'general%';
+------------------+----------------------------------------------+
| Variable_name    | Value                                        |
+------------------+----------------------------------------------+
| general_log      | ON                                           |
| general_log_file | /opt/homebrew/var/mysql/fanweideMac-mini.log |
+------------------+----------------------------------------------+
2 rows in set (0.00 sec)
```



### 查看安全变量值

```shell
SHOW VARIABLES LIKE 'validate_password%';
```

```console
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password.check_user_name    | ON     |
| validate_password.dictionary_file    |        |
| validate_password.length             | 8      |
| validate_password.mixed_case_count   | 1      |
| validate_password.number_count       | 1      |
| validate_password.policy             | MEDIUM |
| validate_password.special_char_count | 1      |
+--------------------------------------+--------+
```

### 密码强度

```
SET GLOBAL validate_password.policy=LOW;    # 低
SET GLOBAL validate_password.policy=MEDIUM; # 中
SET GLOBAL validate_password.policy=STRONG; # 高
```

### 修改密码

1. 将旧密码指控

   ```shell
   use mysql
   ```

   ```shell
   update user set authentication_string = '' where user = 'root';
   ```

   ```shell
   quit
   ```

2. 去除免密码登陆配置

   ```
   skip-grant-tables
   ```

3. 重启

4. 重新登录数据库

   ```shell
   mysql -u root -p
   ```

5. 重置

   ```shell
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'fw123456';
   ```

## deepin or linux 改键

1. 查看与 Ctrl 键有关的映射
  ```shell
  localectl list-x11-keymap-options | grep ctrl:
  ```
2. 交换 lalt 和 lctl
  ```shell
  gsettings set com.deepin.dde.keyboard layout-options '["ctrl:swap_lalt_lctl"]'
  ```


## 技巧
### grep 显示标题行

无标题行

```shell
ps -ef | grep php-fpm
```

```console
501 19144     1   0  3:09下午 ??         0:00.10 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
501 19148 19144   0  3:09下午 ??         1:36.26 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
501 19149 19144   0  3:09下午 ??         1:35.09 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
```

使用命令 `head` 来显示标题行

```sehll
ps -ef | head -1;ps -ef |grep php-fpm
```

```shell
UID   PID  PPID   C STIME   TTY           TIME CMD
501 19144     1   0  3:09下午 ??         0:00.10 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
501 19148 19144   0  3:09下午 ??         1:36.26 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
501 19149 19144   0  3:09下午 ??         1:35.09 /opt/homebrew/opt/php@7.4/sbin/php-fpm --nodaemonize
```

