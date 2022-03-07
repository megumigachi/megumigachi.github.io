---
layout: post
title: ".bash_profile 和 .bashrc"
categories:
  - CS
tags:
  - Random
---

## 虚惊一场

昨天经历了一件十分吓人的事情，差点以为我的 bash 配置文件丢了！好在最后发现是虚惊一场😅。

事情的经过是这样的：由于更新 Rust 下载太慢，我找到了清华镜像站 TUNA 的 Rustup 镜像。以前其实也用过，但是是一次性启用的，这次就想不如直接长期启用了。于是按照指引，输入了下面这条指令：

```bash
$ echo 'export RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup' >> ~/.bash_profile
```

默默关闭了 terminal 窗口，又新开了一个。第一眼看着就有点不对劲，也没细想。但当我敲下指令之后，可怕的事情发生了！

```bash
$ rustup 
rustup: command not found
$ cat ~/.bash_profile
export RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup
```

…………WTF！当时我就慌了，不会吧不会吧，不会我的 `.bash_profile` 被覆盖了吧？？！这时才回头看指令内容 `echo '...' >> ~/.bash_profile`，第一反应是：我竟然真的覆盖了 `.bash_profile`，TUNA 也太坑了吧我要发朋友圈骂一下他……

大脑宕机了一两分钟，然后稍微冷静下来一点以后搜索起了如何恢复被覆盖的文件（没戏）。过了一会儿突然觉得哪里不对，搜了一下发现 `>>` 是 append 啊，不会覆盖啊！！（我本当 bash 苦手）

那么配置没了到底是什么问题呢？5 分钟之后，本盲人终于在 `ls` 的输出中惊讶（惊喜？）地发现了 `.bashrc` ……

## 事情的真相

原来我一直用的 `.bashrc`，没有 `.bash_profile` 。我猜测 `.bash_profile` 的优先级比 `.bashrc` 高，所以上面这个搞笑的事情就发生了。

那么自然就引出了这个问题：`.bash_profile` 和 `bashrc` 有什么区别？

原来事情是这样的：有登录式和非登录式两种 shell。比如 ssh 连接，或者只有命令行的 OS 登录进入的时候，使用的就是登录式 shell。而比如在用 GUI 的 Terminal（xterm）打开的好几个 shell 窗口，使用的就是非登录式 shell。登录式 shell 使用 `profile` ，用来登录后设置环境变量、启动一些服务；而非登录式 shell 使用 `rc`（run command），进行设置别名等功能。我是在 Windows 使用 WSL（Ubuntu），猜测每次打开一个窗口都是单独的登录式 shell。

通常来说，默认有一个 `.bash_profile` ，里面会显式调用 `.bashrc` ，不知道我的 WSL 为啥没有。那为啥我没有 `.bash_profile` 时能成功加载 `.bashrc` ，有了一个空的 `.bash_profile` 以后 `.bashrc` 也没用了呢？我一开始猜测是前者没有就直接调后者了，但后来发现并不是这样，而是另有一个 `~/.profile` ……它里面显式调用了 `.bashrc` 。

其实除了用户级的 `~/.bash_profile` ，`~/.bashrc` ，还有系统级的 `/etc/profile`，和 `/etc/bash.bashrc`。

真正的登录过程其实是这样的：

1. 读取 `/etc/profile` （里面读取了`/etc/bash.bashrc`）

2. 读取 `~/.bash_profile`

- 若不存在，读取 `~/.bash_login`
- 若还不存在，读取 `~/.profile`（里面读取了`~/.bashrc`）

## 这次事件的教训

- 不要不看指令就直接执行……
- dotfiles 有必要备份……

## 参考资料

- [Understanding .bashrc and .bash_profile](https://askubuntu.com/questions/121413/understanding-bashrc-and-bash-profile)
- [关于“.bash_profile”和“.bashrc”区别的总结](https://blog.csdn.net/sch0120/article/details/70256318)
- [How do I restore my .bash_profile?](https://apple.stackexchange.com/questions/26028/how-do-i-restore-my-bash-profile)