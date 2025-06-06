FreeBSD 14.0正式版发布了，我原来一直用`freebsd-update`二进制升级工具。有台机器在2020年初安装FreeBSD 11.0，一直升级到13.2，并没出现什么问题。  
于是胆子肥了，尝试了一把从源代码升级，看看会不会作死，结果差点嗝屁。幸好命大，抢救回来了，真是神灵庇佑。  
于是记个笔记，首先斋戒沐浴，念诵三遍真言：南无加特林菩萨，南无马克沁菩萨，南无阿卡四七菩萨。善哉，善哉！

## 1. 准备好源代码
切换到root用户，将FreeBSD代码树切换到14.0分支。如果一直没有编译过内核和基本系统，则先需要进行以下工作：  

```bash
$ cd /usr/src  #如果没有这个目录就建立一个
$ git clone --branch releng/14.0 https://git.FreeBSD.org/src.git /usr/src 
...
$ git remote --v  #确认源码远程仓正常
origin  https://git.FreeBSD.org/src.git (fetch)
origin  https://git.FreeBSD.org/src.git (push)
```

如果源代码已被正常克隆到本地，使用`git branch`命令，确认本地当前分支为`releng/14.0`：  

```bash
$ git branch
* releng/14.0
```

若`/usr/src`目录下已经有老版本代码，则需要从远程下载14.0分支，并把本机分支切换为`releng/14.0`，如下：  

```bash
$ cd /usr/src
$ git fetch origin releng/14.0 #下载远程分支
$ git checkout releng/14.0     #切换本地分支为releng/14.0
$ git branch     
  releng/13.2
* releng/14.0
```

**吐槽：FreeBSD老喜欢搞些莫名其妙的名字，releng/14.0的意思是14.0-RELEASE版本，stable/14.0的意思是14.0-STABLE版本，main的意思是当前最新开发版本，譬如14-CURRENT。RELEASE和STABLE的主要区别在于：RELEASE的ABI（应用程序二进制接口）会改变，而STABLE不会，10.0-STABLE下编译的程序包在10.1-STABLE下仍然可以运行，不用重新编译。然鹅，这都不重要，问题是为啥RELEASE的代码分支是releng,而不是release？**

切换分支成功后，需要检查一下`/usr/src/UPDATING`这个文本文件的内容，确认里面的更新日志版本号是14.0。

## 2. 编译基本系统和14.0的内核  
FreeBSD下的基本系统叫做`world`，也就是`/bin`,`/sbin`,`/lib`下那些东西，而第三方的软件会放在`/usr/local/bin`、`/usr/local/lib`等下面。这一点反正要比Linux强些，Linux各大发行版的程序各种乱放，就很无语。  
编译基本系统和内核的命令也很简单（`-j40`开关的意思是用40个逻辑内核并行加速）：  

```bash
$ cd /usr/src
$ make -j40 buildworld    #编译基本系统
$ make -j40 buildkernel   #编译新内核
$ make -j40 installkernel #安装新内核
```

发行注记（https://freebsd.org/releases/14.0R/relnotes/）里说最好更新一下`bootstrap`启动器，由于机器不是EFI启动的，且使用了两快硬盘的ZFS文件系统。所以，保险起见，首先查看本机所有硬盘的分区情况：  

```bash
$ gpart show
=>        63  1953525105  da0  MBR  (932G)
          63        1985       - free -  (993K)
        2048     1024000    1  linux-data  (500M)
     1026048  1952497664    2  linux-lvm  (931G)
  1953523712        1456       - free -  (728K)

=>        40  1953525088  da1  GPT  (932G)
          40        1024    1  freebsd-boot  (512K)
        1064     8388608    2  freebsd-swap  (4.0G)
     8389672  1945135456    3  freebsd-zfs  (928G)

=>        40  1953525088  da2  GPT  (932G)
          40        1024    1  freebsd-boot  (512K)
        1064     8388608    2  freebsd-swap  (4.0G)
     8389672  1945135456    3  freebsd-zfs  (928G)

=>        63  1953525105  diskid/DISK-WD-WCAW35925714  MBR  (932G)
          63        1985                               - free -  (993K)
        2048     1024000                            1  linux-data  (500M)
     1026048  1952497664                            2  linux-lvm  (931G)
  1953523712        1456                               - free -  (728K)
```

这表示我的两块双保险的镜像ZFS硬盘分别是`da1`和`da2`，为了拆掉任何一块都能顺利启动，需要执行两次写入，命令如下：  

```bash
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 da1 #第一块硬盘
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 da2 #第二块硬盘
```

理论上，重新启动到新内核，然后干剩下的事情即可。于是，我很自信的输入命令：  

```bash
$ shutdown -r now
```

结果机器愉快的重启，进BIOS，进入FreeBSD的小恶魔LOGO，然后一顿黑文字闪过，卡死在一个错误上。  

```bash
......
Module kernel exists but with wrong version.
```

懵逼了，冷汗都吓出来一斤。各种搜索，貌似有人说需要把原13.2的一些启动加载的模块给取消掉。看似有点道理。但怎么进入系统呢？此时此刻，南无加特林菩萨附体，突然想起来如果编译内核失败旧内核仍然还在。于是`CTRL+ALT+DEL`键重新启动，到进入FreeBSD的`bootloader`界面下按了个`6`，选择启动`kernel`为`kernel.old`。然后忐忑的等待，心中默念，南无马克沁菩萨。  
幸好，顺利进入13.2系统。赶紧编辑`/boot/loader.conf`文件，把`nvidia-modeset_load=“YES”`等等不相干的模块全部用`#`注释掉，为防万一我又重新编译并安装了一次内核：  

```bash
# 等价于 make -j40 buildkernel && make -j40 installkernel
$ cd /usr/src
$ make -j40 kernel
$ shutdown -r now
```

重启机器之后，顺利进入命令行界面，`uname -a`一查，果然内核已经是14.0了，南无阿卡四七菩萨，善哉，善哉。  

## 3. 安装新的基本系统并更新配置文件
既然新内核顺利启动，那么剩下的就很顺利成章：  

```bash
$ etcupdate -p
$ cd /usr/src
$ make -j40 installworld
$ etcupdate -B
......
```
等等，又出错了，提示配置文件冲突，这TM的，只能一个一个地查看：  

```bash
$ etcupdate resolve
......
```

结果发现主要是/etc/master.passwd什么什么的几个文件冲突，但是我一想，反正有神灵保佑，怕个锤子，一律`tf`接受新版本配置文件。然后重启。  
然后悲剧发生了，进去一看，root不需要密码了，原来的用户全消失了。查了文档才知道，貌似14.0默认的root用户shell变为`sh`，配置文件的规则发生了不知道啥的改变。  
不过问题不大，先`passwd root`重新设定一下root密码，然后用`addusr`命令重新添加一下用户名和密码，且把该用户的`home`目录关联为已有的同名目录。重启之后一切正常。虽然又是一斤冷汗，然鹅谁叫我有神灵眷顾呢。南无加特林菩萨，南无马克沁菩萨，南无阿卡四七菩萨。善哉，善哉！  


**小贴士：etcupdate resolve命令对有冲突的配置文件会一个一个询问，还是要仔细看看哪里有不同的，万一哪天三大菩萨开会去了，没顾得上保佑你，那可就完蛋了，宁肯`p`（保留上个系统的配置，以后再处理）。也不要`tf`，切记，切记！！！**   

## 4. 清除过时文件和库

顺利重启，确认系统版本、内核版本，各种正常之后，检查系统中过时的文件和库：  

```bash
$ cd /usr/src
$ make check-old
$ make delete-old
......
```

然后，FreeBSD告诉我什么叫做“安全”。一堆文件，每个都要问我是否确认删除。话说咱都闯过刚才那个大风大浪了，还怕这个？问啥？问就是删除啊！难道一万个文件我要按一万个`y`么？怒`CTRL+z`退出。等等，批量回答`yes`的命令是啥来着？貌似是`yes`，加个管道，如下：  

```bash
$ yes | make delete-old
......
```

居然提示有几个目录删不掉，再`make check-old && make delete-old`一次，诡异的是这回OK了。好吧，很诡异。不过南无加特林菩萨，问题不大，蛤蛤！

**小贴士：查了FreeBSD文档才知道，批量强行删除的命令是`make -DBATCH_DELETE_OLD_FILES delete-old`，又是一斤冷汗。**   

然后再删除过时的库文件，命令如下：  


```bash
$ cd /usr/src
$ make check-old-libs
$ make delete-old-libs
......
```

## 5. 重新安装pkg以及其它软件  

因为我的FreeBSD是RELEASE版，大版本升级后不满足ABI兼容，所以上一个版本已经安装的所有非基本系统的软件包都需要重新安装。命令为：  

```bash
$ pkg bootstrap -f  #重新从远程源安装pkg程序
$ pkg update
$ pkg upgrade
......
```

更新了1800多个软件包，顺利完工。把`/boot/loader.conf`文件里注释掉的内容恢复，然后重启（在此之前我还特地从`ports`又安装了一遍`x11/nvidia-driver`驱动，阿弥陀佛）。切换到普通用户，`startx`顺利进入愉快的`KDE Plasma`桌面。完美。

## 6. 收尾及吐槽  

剩下的工作就是把`/usr/src`的老版本分支给删了，`git branch -d releng/13.2`即可。当然，土豪随意。  
话说虽然搞成了，但要不是三大菩萨保佑，恐怕又得下载一个ISO刻盘重新安装了，冷汗again。  
看来Unix血统的操作系统有一说一，全是各种奇怪问题，各种坑。微软真是一家伟大的公司，试想windows update要是出现什么ABI兼容和kernel版本错误的问题，那就算南无阿帕奇菩萨都保佑不了你。  
不禁想起一个笑话，话说有个Linux高手号称“运维之霸”，每删除一个文件都需要自己确认一遍。众所周知，这个功能在`rm`命令里面要用`-i`参数。于是有一天他自信的在某个控制台下输入：  

```bash
rm * -i
```

结果，这台FreeBSD服务器给他一个回答：  

```bash
-i: No such file or directory
```

惊不惊喜？






