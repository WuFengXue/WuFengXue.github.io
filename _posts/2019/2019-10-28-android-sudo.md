---
layout: post
title: 'android 下的 sudo 命令'
subtitle: '使用 su -c 命令实现 sudo 命令的效果'
date: 2019-10-28
categories: 技术
cover: '../../../assets/img/post_bg/20191019.jpeg'
tags: android
---

## 1. 执行 root 命令的一般流程

* 1、执行 su 命令，切换到 root 用户
* 2、执行目标命令

## 2. sudo 命令
&#160; &#160; &#160; &#160;[sudo](https://baike.baidu.com/item/sudo/7337623?fr=aladdin) 是 linux 系统的一个特殊命令，可以让普通用户直接执行 root 命令（无需先切换到 root 用户）。

&#160; &#160; &#160; &#160;一般用法：sudo cmd

&#160; &#160; &#160; &#160;可以通过执行 sudo -h 查看具体用法，如下：
```doc
$ sudo -h
sudo - execute a command as another user

usage: sudo -h | -K | -k | -V
usage: sudo -v [-AknS] [-g group] [-h host] [-p prompt] [-u user]
usage: sudo -l [-AknS] [-g group] [-h host] [-p prompt] [-U user] [-u user]
            [command]
usage: sudo [-AbEHknPS] [-C num] [-g group] [-h host] [-p prompt] [-u user]
            [VAR=value] [-i|-s] [<command>]
usage: sudo -e [-AknS] [-C num] [-g group] [-h host] [-p prompt] [-u user] file
            ...

Options:
  -A, --askpass               use a helper program for password prompting
  -b, --background            run command in the background
  -C, --close-from=num        close all file descriptors >= num
  -E, --preserve-env          preserve user environment when running command
  -e, --edit                  edit files instead of running a command
  -g, --group=group           run command as the specified group name or ID
  -H, --set-home              set HOME variable to target user's home dir
  -h, --help                  display help message and exit
  -h, --host=host             run command on host (if supported by plugin)
  -i, --login                 run login shell as the target user; a command may
                              also be specified
  -K, --remove-timestamp      remove timestamp file completely
  -k, --reset-timestamp       invalidate timestamp file
  -l, --list                  list user's privileges or check a specific
                              command; use twice for longer format
  -n, --non-interactive       non-interactive mode, no prompts are used
  -P, --preserve-groups       preserve group vector instead of setting to
                              target's
  -p, --prompt=prompt         use the specified password prompt
  -S, --stdin                 read password from standard input
  -s, --shell                 run shell as the target user; a command may also
                              be specified
  -U, --other-user=user       in list mode, display privileges for user
  -u, --user=user             run command (or edit file) as specified user name
                              or ID
  -V, --version               display version information and exit
  -v, --validate              update user's timestamp without running a command
  --                          stop processing command line arguments
```

## 3. android 下的 sudo 命令
&#160; &#160; &#160; &#160;android 虽然是基于 linux 系统，但本身并没有提供 sudo 命令，而是将该功能集成到 su 命令里面。

&#160; &#160; &#160; &#160;我们可以通过执行 su -c cmd 来实现 sudo 的效果。

&#160; &#160; &#160; &#160;例如，在 adb shell 下执行 su -c id 命令，返回的用户信息是 root，而直接执行 id 命令，返回的则是 shell：

```shell
shell@hammerhead:/ $ su -c id
uid=0(root) gid=0(root) context=u:r:init:s0
```

```shell
shell@hammerhead:/ $ id
uid=2000(shell) gid=2000(shell) groups=1003(graphics),1004(input),1007(log),1009(mount),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats) context=u:r:shell:s0
```

## 4. android 下的 su
&#160; &#160; &#160; &#160;通过执行 su \-\-help 命令，可以查看 su 的具体用法：

```shell
shell@hammerhead:/ $ su --help
SuperSU v2.79 (ndk:armeabi-v7a) - Copyright (C) 2012-2016 - Chainfire

Usage: su [options] [--] [-] [LOGIN] [--] [args...]

Options:
  -c, --command COMMAND        pass COMMAND to the invoked shell
  -cn, --context CONTEXT       switch to SELinux CONTEXT before invoking
  -h, --help                   display this help message and exit
  -, -l, --login               pretend the shell to be a login shell
  -m, -p,
  -mm, --mount-master          connect to a shell that can manipulate the
                               master mount namespace - requires su to be
                               running as daemon, must be first parameter
  -mns, --mount-namespace PID  enter mount namespace used by PID
  --preserve-environment       do not change environment variables
  -s, --shell SHELL            use SHELL instead of the default detected shell
  -v, --version                display public version and exit
  -V                           display internal version and exit

Usage#2: su LOGIN COMMAND...

Usage#3: su {-d|--daemon|-ad|--auto-daemon|-r|--reload}
  auto version starts daemon only on SDK >= 18 or
  if SELinux is set to enforcing
  (call only from a root session)

Usage#4: su {-i|--install|-u|--uninstall}
  perform post-install / pre-uninstall maintenance
  (call only from a root session)

Usage#5: su --id pid
  identify eldest parent of pid
  (call only from a root session)
```

&#160; &#160; &#160; &#160;而直接在 linux 系统执行 su \-\-help 命令，则会输出如下内容（以下是 Mac 电脑的输出）：

```shell
$ su --help
su: illegal option -- h
usage: su [-] [-flm] [login [args]]
```

&#160; &#160; &#160; &#160;显然，android 的 su 是定制过的。如果你感兴趣，可以阅读系统源码：[system/extras/su/su.c](http://androidxref.com/6.0.1_r10/xref/system/extras/su/su.c)。


## 5. 总结
&#160; &#160; &#160; &#160;“上帝为你关了这扇门，必会为你再开另一扇门。”

