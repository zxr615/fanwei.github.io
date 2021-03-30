---
title: ssh使用记录
categories: 运维
tags:
  - SSH
date: 2021-01-01 21:55:06
---



## 生成 ssh-key

```console
ssh-keygen -t rsa -C "12345@qq.com"
```



## 免密远程 SSH 登录

1. 将本地 `id_rsa.pub` 添加到本地的 `authorized_keys` 中

2. 将本地 `id_rsa.pub` 添加到服务端 `authorized_keys` 中

3. 服务端配置允许公钥方式登录

    ```shell
      vim /etc/ssh/sshd_config
    
      # 公钥登录
      PubkeyAuthentication yes
      # 公钥文件
      AuthorizedKeysFile .ssh/authorized_keys
      # 密码登录
      PasswordAuthentication yes
    ```

4. 重启 ssh

    ```console
    sudo service ssh restart
    ```
    
5. 配置

    ```console
    cat ~/.ssh/id_rsa.pub
    ```

    ```shell
    # 客户端中的「id_rsa.pub」内容追加进服务端的 「authorized_keys」中
    sudo vim ~/.ssh/authorized_keys
    
    # 免密登录
    SSH fanwei@192.168.0.1
    ```

## 区分多个服务器

客户端：

```
vim ~/.ssh/config

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

免密登录：`SSH ubuntu`



## PhpStorm Terminal 中配置快捷登录 ssh

`Settings -> Tools -> Terminal`

![](https://cdn.jsdelivr.net/gh/zxr615/md-images/images/2020/image-20200830204459428.png)

## ref
https://www.howtoing.com/how-to-set-up-ssh-keys-on-ubuntu-1804