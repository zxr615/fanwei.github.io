---
title: brew 安装 php
categories: 运维
tags:
  - brew
  - PHP
date: 2021-08-25 11:45:07
---

> brew 仓库已经没有7.2以下版本的仓库，如果需要一下版本仓库则需要增加 php 的 tap 仓库



## php-tap 仓库

[https://github.com/shivammathur/homebrew-php](https://github.com/shivammathur/homebrew-php)



## 问题

- 启动时报错

    ```console
    dyld: Library not loaded: /opt/homebrew/opt/tidy-html5/lib/libtidy.5.dylib
      Referenced from: /opt/homebrew/Cellar/php@7.1/7.1.34_4/bin/php
      Reason: image not found
    ```

    > 使用源码重新编译

    ```console
    brew reinstall --build-from-source php@7.1
    ```

## ref

[https://cloud.tencent.com/developer/article/1662505](https://cloud.tencent.com/developer/article/1662505)

