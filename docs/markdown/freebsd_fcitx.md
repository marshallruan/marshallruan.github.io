折腾了FreeBSD 14.0很久，目前感觉KDE Plasma基本可以满足桌面日常了，唯一不爽的是fctix5中文输入法始终不给力:先前是折腾了很久才能在firefox和chrome上敲中文，但输入的过程一直就不太正常，最早的fcitx设置是这样滴：  

```bash
$ cat ~/.xinitrc

export XIM=fcitx
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
export SDL_IM_MODULE=fcitx

exec ck-launch-session /usr/local/bin/startplasma-x11
```
## 1. 拯救firefox和chrome
以上的环境设定在`startx`进入图形桌面后，只有KDE Plasma的原生程序可以中文输入，其他程序上面输入法都表现得莫名其妙，不是打字反映速度慢，就是各种漏风，字母还没有敲完fctix5就给出了错误的选字。　　

这个毛病在网上一搜一大把讯息，情况基本都是一个人提问，其它好心的老爷让他各种贴`locale -a`的结果，打印`fcitx5-diagnose`诊断程序的各种信息。然后给出各种建议，但我几乎都试了一遍，结果照旧。　　

总是一顿折腾后firefox可以输中文，但chrome和vscode不行。然后，不知道在FreeBSD论坛上看到哪个帖子，把配置文件改成了下面这样，这才让firefox和chrome浏览器以及vscode都支持fcitx5的中文输入。但是打字还是各种漂移漏风，无解。　　

```bash
$ cat ~/.xinitrc

export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fctix/xim 　#这里我也不知道为啥这样
export XMODIFIERS="@im=fcitx"
export SDL_IM_MODULE=fcitx

exec ck-launch-session /usr/local/bin/startplasma-x11
```

但好赖是能输入了，尽管[Arch Wiki](https://wiki.archlinux.org/title/Fcitx5)上写了一堆各种解决方案，但问题始终没有解决，以至于在相当长一段时间内，我都把这台电脑当英文机器。　　

这几天升级内核后，我又手贱把windows下的emacs配置和日程迁移到这台FreeBSD下，结果发现emacs的中文输入居然神奇一般的正常了，当时的软件还是2024Q1第一季度的版本，emacs的版本号是29.1，这哪说理去？　　

然而，我毕竟还是太浅薄。到2024Q2的二进制包发布之后，`pkg upgrade`一升级就马上出问题：突然之间chrome和emacs的中文输入统统不行了－在chrome下选词框不跟着光标跑，在emacs下干脆不认中文输入。　　

## 2. Nvidia驱动惊魂
没办法，只能网上到处搜索，各种改，各种`pkg install`什么的，然后`pkg update`或`pkg upgrade`什么的什么的。反正是改了一堆配置，装了一堆程序，卸载了一堆程序，最后就是搞不定。本着大力出奇迹的原则，`shutdown -r now`重启看看，然后悲剧了！系统根本起不来，小恶魔图标后直接如下了：　　

```
kernel panic: page error
#1 ..
#2 ..
#3 ..
...
#17     # 没错我记得最多17行，然后就机器键盘就失效了
```

里面一堆神秘的各种奇怪的符号，不要慌，我看有`nvidia-modset`的字样，肯定是他在作妖。话说我早被`nvidia`的驱动程序训出条件反射了好伐？前两天内核升级就报过`kernel version is wrong`的问题。　　

嗯，看起来用老内核启动应该没问题。于是自信满满的按了机箱的电源键，蛮荒式重启，进入启动画面后按`6`切换到`kernel.old`，结果是......一毛一样的错误，靠！这回真悲剧了。再重启，进入`Single Mode`，仍然是一样的错误。再靠一次，这回真真儿的是悲剧了！　　

当时，脑袋瞬间陷入死循环：要进系统必须把`/boot/loader.conf`里面的`nvidia-modeset_load="YES"`注释掉，但是我进不去系统就注释不掉这个，我注释不掉这个我就进不去系统，我回到家才能拿到核酸证明，我拿到核酸证明保安才能放我回家，我回到家才能拿到核酸证明，好吧，这很扯......

幸亏后来突然想起了FreeBSD可以手动`boot`，在启动界面出现小恶魔图标后按`Esc`，然后输入`set module_balcklist="nvidia-modeset"`，回车确认后再输入`boot`，回车。正常启动，我靠，真是非洲老爷子跳高，黑老子一跳。　　

这回可不敢把`nvidia`驱动放`/boot/loader.conf`文件里面了，删除之，然后在`/etc/rc.conf`的最后一行加上`kld_list+="nvidia-modeset"`。重启，仍然正常，哈人得很。　　

## 3. 继续解决中文输入
再在网上一堆搜索，有人说用`LC_CTYPE=zh_CN.UTF-8 emacs`就能正常，然而一看就知道那帮人是Arch仔，我可是会在emacs里用`(getenv LC_CTYPE)`检查环境变量的男人。果然，这招无效。　　

有人说要设定`~/.xprofile`和`~/.pam_envionment`文件，显然，一看就知道这帮家伙是XWayland仔，不用试都知道无效。　　

还有人各种讨论，让别人贴出各种文件，一会什么`emacs --deamon`，一会什么`systemd`，又是`service dbus status`什么的，教人半懂不懂的，网络上充满了快活的空气。　　

等等，`dbus`，我靠，这个破玩意我咋没注意？秒懂，马上改`~/.xinitrc`文件，如下：　　

```bash
$ cat ~/.xinitrc

export XIM=fcitx
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
export SDL_IM_MODULE=fcitx

exec ck-launch-session dbus-launch --exit-with-sessions /usr/local/bin/startplasma-x11
```

没错，前面几行恢复到一开始的设定，最后一行启动KDE Plasma桌面环境时候增加`dbus-launch`选项。退出图形界面，再重新`startx`。我靠，我靠靠靠，`firefox`和`chrome`上中文输入流畅了，再也不漏风了，跑得比windows的中文输入还顺溜！牛逼，震声！！！再打开`emacs`，中文输入还是不行！傻逼，关灯，吃面！　　

## 4. 发帖求助和灵光一闪
万般无奈，只能可耻的在emacs-china论坛的[Emacs GUI不能输入中文](https://emacs-china.org/t/topic/1271/83)帖子下求助。里面好多Arch仔，于是我就很恐惧。但也只能学圆嘟嘟＂掉阿妈，硬顶上＂，弱弱的在72楼弱弱的问了一贴。这个帖子里面几乎都是被emacs和fcitx5折腾疯的各种linux仔，虽然看起来都很礼貌，但不知道为啥我能闻到他们深深的怨念。　　

本来不指望能有什么靠谱建议的，甚至都不期待有人回复。因为作为一个孤独的FreeBSD仔，空谷里四下无人才是2024的社区环境，更何况楼上的最后一个帖子还是2022年发的。然鹅居然有人回我了（奇迹），说是要装一下`xorg-font-misc-otb`包。我微微一笑，一看就是Arch仔，FreeBSD下没有这个包，装`Xorg`的时候会默认装上所有`misc`字体，为此还特地重新检查了一遍，根本不是字体的问题。　　

都快绝望了，这帮人不是让你检查`LC_CTYPE`就是检查`fctix5-diagnose`，要么就让你装`misc`字体，要么就是让你用各种姿势改`GTK_IM_MODULE`和`XIM`。不靠谱好伐？　　

狗日的升个级就不行了，为啥！等等，升级？看了一下emacs版本，29.3。所以`pkg`的二进制包干了啥？进`/usr/port`看看，会不会这帮货又改了`Makefile`了？　　

```bash
$ cd /usr/port/editor/emacs 
$ make config
```

果然，默认的配置把`CARIO`图形库打开了，于是`XFT`和`OTF`支持啥的都关了（`CARIO`和`XFT`你只能二选一），是不是这个原因？试着把配置改成图1的样子，然后`make deinstall`，再`make && make install`。　　

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 1px 2px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/emacs_make_config.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图1 修改后的编译选项（红框内是主要改动） </div>
</center>

启动emacs，喜大普奔，苍天啊，大地啊，这哪位`port`维护者在`Makefile`里面乱放屁啊？正常了！　　

## 5. 总结
虽然，Arch仔就知道不停的`LC_CTYPE`和`fcitx5-diagose`还有字体什么的叨B叨，但还是要感谢那位不吝给我回复的glgl-schemer同学，话说，用`scheme`语言的仔不是看不起`elisp`么？总之，说谢谢不犯法，感谢感谢！　　

雀食，emacs在linux和freebsd下不能用fcitx5的原因无非这几个：　　

1. `LC_CTYPE`没有设定为`zh_CN.UTF-8`；
2. `XIM`和`GTK_IM_MODULE`等等没有正确的设置，这个在FreeBSD下照抄我的即可，但是不要用`~/.profile`和`~/.pam_envionment`，和这些就没毛关系；
3. linux下`dbus`在图形界面下默认是开的，而万恶的FreeBSD需要要手动开启，参考前面`~/.xinitrc`文件的最后一行；
4. 确认系统安装了`xorg-misc`相关字体，且字体路径在`X11`启动时已经正确的导入了；
5. 如果都没有问题，且fctix5在其他程序上正常，但就是不支持`emacs`，那么肯定是`emacs`安装包的编译选项不对，坑爹啊！　　

最后，`nvidia`的驱动经常会搞死FreeBSD，若不能启动，要手动`module_blacklist`命令行`boot`。　　

最后的最后，FUCK NVIDIA！！！

