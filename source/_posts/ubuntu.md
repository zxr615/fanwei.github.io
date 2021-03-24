---
title: ubuntu安装node
categories: 运维
tags:
  - node
  - Linux
date: 2021-03-18 23:33:29
---

ubuntu 安装node

https://github.com/nodesource/distributions

```
# Using Ubuntu
curl -fsSL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs
```



将子模块 也 `clone` 下来

```console
git clone --recurse-submodules https://github.com/zxr615/zxr615.github.io.git
```



clone 子模块 

```console
git submodule update --init --recursive
```

