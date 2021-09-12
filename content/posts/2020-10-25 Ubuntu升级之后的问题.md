---
title: Ubuntu升级之后的问题
date: "2020-10-25T16:03:00.000Z"
description: "Ubuntu升级之后的问题"
---

自己的系统本来在Ubuntu18.04运行得好好的，出于好奇，升级到20.04，过程倒还顺利，结果登陆的时候原有的账户不见了，只有装新系统时候创建的那个用户，登陆的界面倒是还提供了一个 “not listed”的选项，结果每次登陆的时候，黑屏后还是进不去，每次还是退回到登陆界面。以下是我解决的几个问题，记录如下:

1. 登陆不进去的时候，要尝试命令行是否能够登陆。CTRL+ALT+F3后在命令行登陆的时候，说我的zsh没有安装，之前系统配置了zsh，可是升级的时候没有了，重新在命令行用`apt install zsh`后就没有问题了，然后在命令行下面用`startx`可以顺利地进入图形界面，这说明没有什么大问题。 

2. 先是尝试解决这个重复登陆的问题，用了下面这个帖子[letsfoss](https://letsfoss.com/ubuntu-20-04-login-loop-fix/)，结果所有5种方法都试过了，重启了无数次，结果没有用。

3. 放弃重复登陆问题，尝试解决老账户不在登陆界面显示的问题。找到这个[帖子](https://askubuntu.com/questions/1234452/ubuntu-20-04-user-not-listed-to-login)，事实证明，这个是关键，也就是如下步骤：

>sudo su we need to be superuser

>cd /var/lib/AccountsService/users here we go to the service that shows users, issue is here (at least in my case)

>you should see multiple files, one for each of your users, including the old one that is not showed in your screen. In this file check the value of SystemAccount. If it's true, then this is the issue: it should be false.

>if it does not help you, simply check differences between users files, the one that works with ones that don't. Also check that Icon path is correct.

>don't forget to exit su, and reboot after each try. Good luck, it took 3h digging before a fixed my issue.

在我的情况，我不能登陆的账户是下面这样的：

```
[User]
Session=
FormatsLocale=en_CA.UTF-8
XSession=unity
Icon=/home/aaron/.face  //这个文件是不存在的，直接删掉，也就是等于号后面留空白即可
SystemAccount=true  //这里需要改成false

[InputSource0]
xkb=us

[InputSource1]
ibus=libpinyin

[InputSource2]
ibus=pinyin
```

经过上面两处更改后，重启后顺利登陆。 

在这个过程中，需要用到root用户权限，如果忘记，可以使用 `sudo passwd root`来重新设置，具体如下![所示](https://www.cyberciti.biz/media/new/faq/2017/07/How-to-change-root-password-in-ubuntu-Linux.jpg)

4. 升级之后Search不再有效。

[解决方法](https://askubuntu.com/questions/1262114/search-your-computer-sorry-there-is-nothing-that-matches-your-search)如下：

>sudo apt install ubuntu-unity-desktop

>sudo apt remove gnome-shell

>sudo dpkg-reconfigure lightdm

>sudo reboot

问题的根源在于升级之后gdm3变成了lightgdm。需要改回去就可以了。
