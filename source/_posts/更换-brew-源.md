---
title: 更换 brew 源
categories: 运维
tags:
  - mac
  - brew
date: 2021-05-06 10:34:35
---

> 中科大镜像

替换 Homebrew

```console
git -C "$(brew --repo)" remote set-url origin https://mirrors.ustc.edu.cn/brew.git
```

替换 Homebrew Core

```console
git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
```

替换 Homebrew Cask

```console
git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git
```

zsh

```bash
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc
```

```shell
source ~/.zshrc
```

更新brew

```console
brew update
```

> error: Not a valid ref: refs/remotes/origin/master

如遇到 error 替换为

```console
brew update --verbose
```

