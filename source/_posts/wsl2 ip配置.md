---
title: wsl2 ip配置
categories: 运维
tags:
  - wsl2
  - Linux
  - Ubuntu
date: 2021-3-24 22:06:43
---

## 自动获取wsl2ip

> 每次重启获取内部ip 并且会写入到 windows 的 hosts 中

[https://link.zhihu.com/?target=https%3A//github.com/shayne/go-wsl2-host/](https://link.zhihu.com/?target=https%3A//github.com/shayne/go-wsl2-host/)

安装脚本

```
.\wsl2host.exe install
```

输入电脑登录的帐号密码

```shell
Windows Username: 用户名
Windows Password: 密码
```

启动

```shell
.\wsl2host.exe start 
```

失败的话检查服务

`win + R` 输入 `services` 

<img src="https://raw.githubusercontent.com/zxr615/md-images/master/images20210324215158.png" alt="image-20210324215157987" style="zoom:50%;" />



双击可编辑密码，我这里必须要在这里设置一下密码，命令行设置的密码无效

<img src="https://raw.githubusercontent.com/zxr615/md-images/master/images20210324215358.png" alt="image-20210324215358692" style="zoom:50%;" />

## windows端口转发

增减端口转发

> listenport 为 windows 需要监听的端口，connectport 是 WSL2 你的服务端口，如果不用转发了可以执行删除命令

```console
netsh interface portproxy add v4tov4 listenport=8080 connectaddress=ubuntu.wsl connectport=8080 listenaddress=* protocol=tcp
```

删除端口转发

```console
netsh interface portproxy delete v4tov4 listenport=8080 protocol=tcp
```

