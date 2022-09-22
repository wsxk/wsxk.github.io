---
layout: post
title: "ubuntu安装zsh"
date:   2022-3-9
tags: [software_build]
comments: true
author: wsxk
---

## 1.安装zsh

    sudo apt install zsh #安装zsh
    chsh -s /bin/zsh  #设置成默认shell
    exec zsh        # 重启终端

## 2.安装Oh-My-Zsh

Oh-My-Zsh是zsh的拓展，可以为zsh选择你喜欢的shell风格

    wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh

## 3.配置主题

我选的是Powerlevel10k主题

    git clone --depth=1 https://gitee.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

    sudo vim ~/.zshrc

编辑.zshrc 中的ZSH_THEME参数

ZSH_THEME="powerlevel10k/powerlevel10k"

    source ~/.zshrc