---
title: samba使用记录
categories: 运维
tags:
  - Samba
date: 2020-11-11 23:57:16
---



## 资料

[鸟哥的私房菜-第十六章、文件服务器之二： SAMBA 服务器](http://cn.linux.vbird.org/linux_server/0370samba_2.php)



## 安装

```
# samba
sudo apt install samba

# samba-client
sudo apt install smbclient
```



## 免密配置

```shell
[global]
   workgroup = WORKGROUP
   server string = %h server (Samba, Ubuntu)
   dns proxy = no
   log file = /var/log/samba/log.%m
   max log size = 1000
   panic action = /usr/share/samba/panic-action %d
   server role = standalone server
   passdb backend = tdbsam
   obey pam restrictions = yes
   unix password sync = yes
   passwd program = /usr/bin/passwd %u
   passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
   pam password change = yes
   map to guest = bad user
   usershare allow guests = yes
   # 创建文件用户
   force user = fanwei
   # 创建文件组
   force group = fanwei
   create mask = 0777
   directory mask = 0777


[project]
    comment = Project
    path = /home/fanwei/project
    writable = yes
    browseable = yes
    guest ok = yes
```

`sudo systemctl restart smbd.service`

## 测试语法

`testparm`

## 测试连接

### smbclient

```shell
smbclient -L //127.0.0.1
```

### windows 

在文件资源中输入`\\ip`

## 备注

- 如果访问不了、无法写入，检查文件夹权限

