---
title: xdebug & phpstorm 调试
categories: PHP
tags:
  - xdebug
  - php
date: 2021-04-06 11:16:54
---

## 安装

[https://xdebug.org/docs/install](https://xdebug.org/docs/install)



## 添加xdebug扩展

php.ini

```console
; xdebug
zend_extension="xdebug.so"
xdebug.mode="debug"
xdebug.client_host="127.0.0.1"
xdebug.client_port=9100
```



## 设置phpstorm

<img src="https://cdn.jsdelivr.net/gh/zxr615/md-images/images/2020image-20210406112119754.png" alt="image-20210406112119754" style="zoom: 25%" />

<img src="https://cdn.jsdelivr.net/gh/zxr615/md-images/images/2020image-20210406112400893.png" alt="image-20210406112400893" style="zoom: 25%;" />

