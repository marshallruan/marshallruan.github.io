安装了Freebsd-12.1-RELEASE，发现硬盘太小，且有坏道，想用新盘替换。
原系统的硬盘信息为:  
```shell
> gpart show /dev/diskid/DISK-4MT0PQB6
=>       40  312581728  diskid/DISK-4MT0PQB6  GPT  (149G)
         40       1024                     1  freebsd-boot  (512K)
       1064  304086016                     2  freebsd-zfs  (145G)
  304087080    8388608                     3  freebsd-swap  (4.0G)
  312475688     106080                        - free -  (52M)
```
在我的计算机上，老盘是DISK-4MT0PQB6，新盘是DISK-9QF4ATCW。
一般FreeBSD系统用`/dev/ada0`和`/dev/ada1`之类的路径表示硬盘。但我的机器支持硬盘热插拔，且经常因手贱各种捣鼓，硬盘位可能会改变，所以用`/dev/diskid/...`的方式更保险。
当然还可以用disklabel命令给每块硬盘取个名字并且贴上标签。因暂时我只有两块硬盘，还无需简单问题复杂化。  

现在有一块新硬盘(DISK-9QF4ATCW)，需格掉原来的windows分区，建立和DISK-4MT0PQB6一样的分区表。
根分区和这块盘做成mirror即可自动同步。
同步完毕后对新盘的freebsd-boot分区写入启动项。
最后对新磁盘扩容。

由于老磁盘的freebsd-swap在第3个分区，freebsd-zfs和free的磁盘空间不连续，为扩容方便，新磁盘的分区格式最好是像下面这样。
ZFS分区和老盘的大小一样，所以在复制完毕后还要对新盘的pool和分区都进行扩容。
```shell
1  freebsd-boot  (512K)
2  freebsd-swap  (4.0G)
3  freebsd-zfs  (145G)
    - free -  (84G)
```

**小贴士：建议安装FreeBSD时把可能需要扩容的freebsd-zfs格式分区放在分区表的最后。**

## 1. 新硬盘建立分区表
### 1.1 清除新盘的分区

```
> gpart destroy -F /dev/diskid/DISK-9QF4ATCW
diskid/DISK-9QF4ATCW destroyed
```
参数`-F`为强行抹去分区表，如果硬盘原来被挂载过，不加此参数可能会显示Devise Busy的错误。
### 1.2 设置磁盘为GPT分区表
```
> gpart create -s GPT /dev/diskid/DISK-9QF4ATCW
diskid/DISK-9QF4ATCW created
```
安装一个保护性的MBR列表，这样无GPT支持的BIOS也能从本磁盘启动：

```
> gpart bootcode -b /boot/pmbr /dev/diskid/DISK-9QF4ATCW
bootcode written to diskid/DISK-9QF4ATCW
```
### 1.3 建立freebsd-boot分区
该分区大小为1024K:
```
> gpart add -b 40 -s 1024 -t freebsd-boot /dev/diskid/DISK-9QF4ATCW
diskid/DISK-9QF4ATCWp1 added
> gpart bootcode -p /boot/gptboot -i 1 /dev/diskid/DISK-9QF4ATCW
partcode written to diskid/DISK-9QF4ATCWp1
```
### 1.4 建立swap分区
为和老磁盘一致，这里也建立一个4GB的swap分区，但完全可以分得更大或更小:
```
> gpart add -s 4G -t freebsd-swap /dev/diskid/DISK-9QF4ATCW
diskid/DISK-9QF4ATCWp2 added
```
### 1.5 建立用于同步根目录的zfs分区
```
> gpart add -s 145G -t freebsd-zfs /dev/diskid/DISK-9QF4ATCW
diskid/DISK-9QF4ATCWp3 added
```
新盘分区完毕之后，检查是否正确：

```
> gpart show /dev/diskid/DISK-9QF4ATCW
=>       40  488397088  diskid/DISK-9QF4ATCW  GPT  (233G)
         40       1024                     1  freebsd-boot  (512K)
       1064    8388608                     2  freebsd-swap  (4.0G)
    8389672  304087040                     3  freebsd-zfs  (145G)
  312476712  175920416                        - free -  (84G)
```
这里第3个分区的大小和老磁盘的第二个分区一样，作为操作系统根分区的镜像。
理论上新盘的根分区也可以和老盘不一致，这样zpool同步完成后无需进行resize操作，不过考虑到“在unix下什么糟糕的情况都有可能发生”，我采用最保险的策略。

## 2. 存储池（zpool）同步
### 2.1 检查操作系统zpool
如果Freebsd安装在ZFS分区上，默认会有一个名为root的zpool，而在线镜像复制需将新盘作为mirror加入root存储池。
```shell
> zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
root   144G   117G  27.0G        -         -    39%    81%  1.00x  ONLINE  -
```
查看root存储池的状态：
```shell
> zpool status root
  pool: root
 state: ONLINE
  scan: resilvered 117G in 0 days 01:00:06 with 0 errors on Wed May 20 07:56:42 2020
config:

	NAME                      STATE     READ WRITE CKSUM
	root                      ONLINE       0     0     0
	  diskid/DISK-4MT0PQB6p2  ONLINE       0     0     0

errors: No known data errors
```
以上显示`pool: root`下只有老磁盘的第2分区。

### 2.2 增加硬盘镜像
将新磁盘的第3分区作为镜像attach到root存储池里：  
```shell
> zpool attach root /dev/diskid/DISK-4MT0PQB6p2 /dev/diskid/DISK-9QF4ATCWp3
```
以上命令把新磁盘的第3分区和老磁盘的第2分区作为镜像。等价于对操作系统根分区作RAID-1，不同的是ZFS无需RAID卡，所有过程都是自动的软操作。

### 2.3 查看zpool状态
```shell
> zpool status root
  pool: root
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed May 20 08:14:26 2020
	117G scanned at 1.48G/s, 849M issued at 10.7M/s, 117G total
	840M resilvered, 0.71% done, 0 days 03:04:58 to go
config:

	NAME                        STATE     READ WRITE CKSUM
	root                        ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    diskid/DISK-4MT0PQB6p2  ONLINE       0     0     0
	    diskid/DISK-9QF4ATCWp3  ONLINE       0     0     0

errors: No known data errors

```
以上表明两块硬盘的两个根分区已建立镜像模式，数据会自动同步到新盘。

### 2.4 继续查看zpool状态
```shell
> zpool status root
 pool: root
 state: ONLINE
  scan: resilvered 117G in 0 days 01:00:53 with 0 errors on Wed May 20 03:01:06 2020
config:

	NAME                        STATE     READ WRITE CKSUM
	root                        ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    diskid/DISK-4MT0PQB6p2  ONLINE       0     0     0
	    diskid/DISK-9QF4ATCWp3  ONLINE       0     0     0

errors: No known data errors
```
以上显示已同步117G的数据，同步完成，无错误发生。此时可用粤语吼一声“好彩”...... 


### 2.5 在新磁盘启动分区写入启动代码

```shell
> gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 /dev/diskid/DISK-9QF4ATCW 
partcode written to diskid/DISK-9QF4ATCWp1
bootcode written to diskid/DISK-9QF4ATCW
```
参数`-i 1`表示启动代码写入DISK-9QF4ATCW的第1分区，即freebsd-boot分区。

### 2.6 在root存储池里删掉老磁盘
```shell
> zpool detach root /dev/diskid/DISK-4MT0PQB6p2
```
若命令行无错误信息提示，此时即可拔掉老磁盘（确认你的机器支持热插拔）。

## 3. 新磁盘扩容
### 3.1 将freebsd-root分区扩容
由于新磁盘的ZFS根分区在最后，分区扩容非常方便:
```shell
> gpart resize -i 3 /dev/diskid/DISK-9QF4ATCWp3
diskid/DISK-9QF4ATCWp3 resized
> gpart show /dev/diskid/DISK-9QF4ATCW
=>       40  488397088  diskid/DISK-9QF4ATCW  GPT  (233G)
         40       1024                     1  freebsd-boot  (512K)
       1064    8388608                     2  freebsd-swap  (4.0G)
    8389672  480007456                     3  freebsd-zfs  (229G)
```
### 3.2 将root存储池扩容
虽然新磁盘的容量已扩大为229G，但操作系统的root存储池仍然是老系统的114G，如下：
```shell
> zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
root   144G   117G  26.7G        -         -    40%    81%  1.00x  ONLINE  -
```
所以需要执行以下命令把root存储池扩容:
```shell
> zpool online -e root /dev/diskid/DISK-9QF4ATCWp3
```
用以下命令检查是否扩容成功:  
```
> zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
root   228G   117G   111G        -         -    25%    51%  1.00x  ONLINE  -
```

## 总结
在ZFS下在线扩容或更换硬盘非常简单。硬盘分区应尽量把数据区放在分区表的最后。  
以上更换硬盘的方法是利用ZFS的镜像功能复制了整个根分区，然后在新硬盘启动分区中写入启动代码。





