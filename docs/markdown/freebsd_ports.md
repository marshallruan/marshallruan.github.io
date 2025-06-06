FreeBSD操作系统有两种软件包管理方式：二进制的pkg方式和源代码的ports方式。  
pkg方式是从服务器下载二进制安装包到本地安装，和Redhat系Linux的yum或Debian系Linux的apt类似。常用命令如下： 

```bash
#更新二进制软件包索引
$ sudo pkg update    
#更新本机安装的所有软件包
$ sudo pkg upgrade
#安装某个软件包，比如firefox，它会自动安装所有依赖（也可pkg add）
$ sudo pkg install firefox
#删除某个软件包，比如firefox，它会删除依赖此包的软件（也可pkg delete）
$ sudo pkg remove firefox
#查看某个软件的安装信息（如firefox）
$ pkg info firefox
#自动删除所有无第三方依赖的被依赖包（慎用，有时候循环依赖，麻烦）
$ sudo pkg autoremove
#删除已下载的缓存二进制安装源(.txz文件,一般位于/var/cache/pkg/目录下)
$ sudo pkg clean

```

源码级的ports编译安装方式是另一种非常经典的Unix软件包管理系统，MacOS上的`homebrew`和Gentoo Linux的`portage`都模仿了这一设计。  
使用ports方式需要下载软件和其所有依赖库的源代码在本机上编译安装。和pkg的二进制安装方式不同，它可以根据本机的CPU架构优化编译软件，相当适合强迫症患者。  

**小贴士：用源代码编译安装虽然会尽量挖掘本机性能，但编译是一种非常耗时和耗能的事情。从环保和时效考虑，建议像Gnome3、Kde-plasma5这样的大型软件直接用pkg二进制安装。**  

ports系统并没有直接存储软件的源代码，否则机器的硬盘早爆了。它只是一个`Makefile`、补丁（`patch`）和软件描述文件的集合。FreeBSD用这个集合来规定如何编译和安装软件，每个软件对应一个`port`，所有支持的软件对应`ports`，默认存放路径为`/usr/ports/`。

## 1. 下载ports到你的计算机
刚刚安装好的FreeBSD基本系统并没有在硬盘上自动建立ports，但提供了一个叫做`portsnap`的命令，它可以连接FreeBSD网站，核对安全密钥，并下载最新的ports到本机上。  
下载ports的压缩快照（snapshot）到本机（存放在/var/db/portsnap/目录下）的命令是：  

```bash
$ sudo portsnap fetch
```
下载完毕后，解压此快照（默认解压到/usr/ports/）的命令是：  

```bash
$ sudo portsnap extract
```

从今以后，手动更新ports系统的命令是：  

```bash
$ sudo portsnap fetch
$ sudo portsnap update
```

`portsnap`命令支持参数的顺序调用，用户可以这样使用它：  

```bash
#第一次下载ports系统到本机
$ sudo portsnap fetch extract
#此后手动更新ports系统
$ sudo portsnap fetch update
```

`/usr/ports/`目录下按软件的不同分类建立子目录，每个软件的目录下只有如何编译并安装它的信息，这些信息被称为骨架（`skeleton`），每个skeleton包含以下文件或子目录：  
&emsp;&emsp;`Makefile`：此软件的编译过程和安装目的地。  
&emsp;&emsp;`distinfo`：此软件需要下载的源码包的名称、校验码等信息。  
&emsp;&emsp;`files/`：此软件需要的补丁包及其他文件。  
&emsp;&emsp;`pkg-descr`：此软件的更详细的信息，用于pkg系统管理。  
&emsp;&emsp;`pkg-plist`：此软件将会安装到本机上的所有文件，卸载时需删除哪些文件。  
有些特殊的软件还会有`pkg-message`或其它文件，详细信息请参考FreeBSD手册。整个`/usr/ports`下没有任何真实的软件源码，只有编译安装时它们才会被自动下载到`/usr/ports/distfiles/`目录。

## 2. 使用ports编译安装软件
首先要求计算机能访问Internet，并且当前用户是`root`，或拥有`root`的用户密码。  
安装一个软件（如`vim`）需在此软件的ports目录下执行make install命令：  

```bash
$ cd /usr/ports/editors/vim
$ sudo make install
```

控制台会提示下载源码包的进度、编译的输出信息及安装信息。编译安装过程会生成一些临时文件，安装成功后可以清空：  

```bash
$ sudo make clean
```
如果你不想在`/usr/ports`的子目录下跳来跳去，可以用`-C`参数避免。首先，你必须在`/usr/port`目录下，然后以上命令会是这样：  


```bash
$ cd /usr/ports/
$ sudo make -C editors/vim install
$ sudo make -C editors/vim clean
$ sudo make -C editors/vim install clean
$ sudo make -C editors/vim reinstall clean
```

**小贴士：可使用`sudo make install clean`一次性完成编译、安装、清空的工作。**

有些软件由很多组件（比如`/usr/ports/www/firefox`），编译时会弹出蓝色文本对话框等待用户编辑配置信息，其依赖包也可能会如此。由于编译大型软件非常耗时，用户不可能一直眼巴巴的等待一个包编译完毕，然后再修改下一个包的配置文件。因此事配置好所有包的编译选项尤为必要。这一过程通常如下：  

```bash
#到firefox的port目录下
$ cd /usr/ports/www/firefox
#首先递归配置好所有包的编译选项
$ sudo make config-recursive
#这样firefox和其所有依赖包的编译选项配置对话框会一个个出现
#设定好所有配置后开始编译，此过程不会有对话框打断，自动完成
$ sudo make install clean

```

## 3. 卸载ports软件及清理临时文件
用ports编译安装的软件可以被pkg系统管理，可使用`pkg delete`或`pkg remove`命令来卸载它们。另外，使用Unix传统的`make uninstall`命令也可卸载它们，以`vim`为例，命令为：  

```bash
$ cd /usr/ports/editors/vim
$ sudo make deinstall
```

长期使用ports系统会占用大量磁盘空间，故而推荐编译安装命令是`make install clean`。但是，FreeBSD为了不重复下载同一源码，下载的源码包都存在`/usr/ports/distfiles/`中且不会被删除。因此，建议定期清理此目录下的文件。比较简单粗暴的方式如下：  

```bash
$ cd /usr/ports/distfiles
$ sudo rm -rf *
```

但这样可能显得不那安全，某些手抖人士可能敲成了`sudo rm -rf ～/*`：）。  
建议安装一个叫做`portupgrade`的软件包（`/usr/ports/ports-mgmt/portupgrade/`），它提供`portsclean`命令安全清除`/usr/ports/distfiles/`下的过期源码，使用方式如下:  

```bash
#删除所有编译中间文件
$ sudo portsclean -C
#删除更新/usr/ports后不被所有软件依赖的源代码
$ sudo portsclean -D
#清空所有本机已通过ports编译安装的软件不依赖的源代码
$ sudo portsclean -DD

```

## 4. 根据软件名搜索ports
在浩如烟海的ports系统中去寻找某个软件的所在目录可以用Unix经典愚蠢的`find`命令。同时，FreeBSD的`make`也提供了`search`参数支持ports搜索，比如搜索`vim`软件包的命令为：  

```bash
#改变当前目录为/usr/ports/
$ cd /usr/ports
$ make search name=vim
```

由于包含`vim`这个字符的内容太多，可以用`grep`命令简化输出：  
```
$ make search name=vim | grep Path:
Path:   /usr/ports/audio/vimpc
Path:   /usr/ports/devel/geany-plugin-vimode
Path:   /usr/ports/devel/geany-plugin-vimode
Path:   /usr/ports/devel/p5-Shell-EnvImporter
Path:   /usr/ports/editors/neovim
Path:   /usr/ports/editors/neovim-qt
Path:   /usr/ports/editors/p5-Vimana
Path:   /usr/ports/editors/py-pynvim
Path:   /usr/ports/editors/py-pynvim
Path:   /usr/ports/editors/rubygem-neovim
Path:   /usr/ports/editors/vim
Path:   /usr/ports/editors/vim-console
Path:   /usr/ports/editors/vim-tiny
Path:   /usr/ports/games/rubygem-vimgolf
Path:   /usr/ports/japanese/jvim3
Path:   /usr/ports/sysutils/vimpager
Path:   /usr/ports/textproc/p5-Text-VimColor
Path:   /usr/ports/www/vimb
```

## 5. 修改/etc/make.conf文件
从源码编译安装软件非常折腾时间，对于很多强迫症来说，这样做的最大理由可能就是为了让软件在本机上跑得比香港记者还快，但需要修改编译器优化参数。  
同时，由于众所周知的原因，在某片960万平方公里的神奇土地上很多网站无法访问，这群全球最大局域网的可怜用户往往需要设定源码镜像站点。  
最后，由于Sun公司搞出来Java这个神奇的、罗嗦的、降低智商的编程语言，软件源码文件大小承指数级上升，多线程下载极有必要。  
以上需求都要修改`/etc/make.conf`文件，详细说明请键入`man make.conf`命令查文档，以下是我机器上的`/etc/make.conf`： 

```bash
#默认使用axel命令分5个线程下载，-a表示只输出简短下载信息而不刷屏
#当然你需要实现安装axel程序，pkg install axel就行了
FETCH_CMD=axel -n 5 -a

DISABLE_SIZE=yes

#中科大的镜像有时候很诡异
MASTER_SITE_OVERRIDE?=\
http://mirrors.ustc.edu.cn/freebsd-ports/distfiles/${DIST_SUBDIR}/\
http://mirrors.163.com/freebsd/ports/distfiles/${DIST_SUBDIR}/\
http://mirrors.aliyun.com/freebsd/ports/distfiles/${DIST_SUBDIR}/\
http://ports1.chinafreebsd.cn/distfiles/${DIST_SUBDIR}/

MASTER_SITE_OVERRIDE?=${MASTER_SITE_BACKUP}

#编译器优化命令：
#   CPU类型，我的CPU是intel志强E5-2697v2是ivybridge的，如果不知道就别设这个
CPUTYPE?=ivybridge
#   -O3解释，不O3不男人。 -pipe子过程用管道而非文件通讯，可缩短编译时间
#   -funroll-loops，这个是个老技巧了，函数的循环展开。其实也提速不了多少
CFLAGS=-O3 -pipe -funroll-loops
COPTFLAGS=-O3 -pipe -funroll-loops
BUILD_OPTIMIZED=YES
```

## 6. 注意事项
有些人很强迫症，每天都要更新一下软件获取心理安慰。有事没事就这样：  

```bash
$ sudo pkg upgrade
```

但是如果你辛辛苦苦的从ports编译安装了一个大型软件（比如Gnome3），现在pkg发现有更新的二进制版本了，它会把原来的ports版本覆盖掉。傻了吧？  

**小贴士：FreeBSD下除了基本系统的安全补丁需要更新之外，其它软件没事别老更新。**

真要更新ports编译安装的软件应该象这样操作：  

```bash
#首先更新ports
$ sudo portsnap fetch update
#然后查找所有用ports安装的但已过期的软件
$ sudo pkg version -l "<"
#你可以一个个make reinstall clean，也可以写个脚本批量重新编译它们
#现在你的ports安装的软件已经是最新的了，就可以放心的更新二进制包了
$ sudo pkg upgrade
```

从源码升级所有ports软件的另一个简单办法是第3节介绍的`portupgrade`程序，它提供了`portsclean`命令清空临时文件，但更重要的功能是更新ports软件包。更新之前，强烈建议用`pkgdb -F`命令扫描已安装的ports并根据提示修复所有不一致的软件包，然后再进行ports软件升级，命令为：  
```bash
# -a参数表示升级所有系统安装的可升级ports，
# -i命令表示每个软件包需要用户交互确认。
$ sudo portupgrade -ai
```

若想单独升级某个特定的软件包（如firefox），需使用以下命令：  

```bash 
# -R参数表示首先升级该软件的依赖包
$ sudo portupgrade -R firefox 
```

假设本机器用pkg方式安装了一些软件，用ports方式安装了其它软件，那么比较推荐的升级方法是：
```bash
#当然如果你是真男人，完全可以直接-a，基和谐佬才用-i
$ sudo portupgrade -ai
#若ports升级成功后再进行二进制安装的软件更新
$ sudo pkg upgrade
```
