[TOC]
# 硬件主机--存储
## 磁盘
- 用UUID挂载磁盘，以免磁盘换到别的机器，位置错误。导致挂载错误。
- 不重启，识别新添加的硬件。`echo "- - -" > /sys/class/scsi_host/host$i/scan`

## 磁盘不能mkfs

- 故障检查:`/dev/sdb1 显然正被系统使用； 取消建立 文件系统 ！`

![soft raid device](../../images/raid-md.png)

```



[root@localhost dev]# mdadm --force --stop /dev/md126
mdadm: --force does not set the mode, and so cannot be the first option.
[root@localhost dev]# mdadm --force --stop 
mdadm: --force does not set the mode, and so cannot be the first option.
[root@localhost dev]# mdadm --force --stop md
mdadm: --force does not set the mode, and so cannot be the first option.



```

## 磁盘不能umount
```bash
fuser -m /home/prodlib # 查看具体是哪些进程在占用目录
cat /proc/13004/stat    # 查看进程当前状态，可看可不看
kill -9 进程号  # 如果还不能杀死，看是否可以kill父进程，如果还不行，只能重启了。
```

## 添加磁盘&&扩展系统中分区容量
#### 文件系统是lvm类型时
- 查看分区大小`df -h`
- 使用`fdisk -l`查看分区大小
  - 磁盘 `/dev/sda：139.6 GB`说明我们已经成功对磁盘进行了扩容，但是需要分区

- 开始分区
```bash
fdisk /dev/sda  ##这里的/dev/sda是我们刚扩容的磁盘
Command (m for help): p ##输入

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    62914559    30407680   8e  Linux LVM 

Command (m for help): n ##输入
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p ##输入
Partition number (3,4, default 3): 3 ##输入 我们创建的分区号为3
First sector (62914560-272629775, default 62914560): 回车默认
Last sector, +sectors or +size{K,M,G} (62914560-272629775, default 272629775): 回车默认

Command (m for help): t 输入
Partition number (1-3, default 3): 3 输入
Hex code (type L to list all codes): 8e 输入
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): w 输入
t xidian]# partprobe 键入
[root@localhost xidian]#
```
- 查看磁盘空间`df -h`

- 发现磁盘容量并没有增加。

- 创建物理卷并加入卷组，扩展逻辑卷。
  - 这里使用`pvs`查看逻辑卷名称
  - 逻辑卷名称为`centos`

```bash
pvs

  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <29.00g    0
```

- 下面创建物理卷并加入卷组
```bash
## 查看vg的name
[root@localhost xidian]# pvs  下面VG对应的值就是卷组名为centos
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <29.00g    0

  ## 创建pv
[root@localhost xidian]# pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created.
## 查看物理卷/dev/sda3大小为100G
[root@localhost xidian]# pvs 
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <29.00g      0
  /dev/sda3         lvm2 ---  100.00g 100.00g

## 扩展sda3容量
[root@localhost xidian]# vgextend centos /dev/sda3
  Volume group "centos" successfully extended
[root@localhost xidian]# lvextend -l +100%FREE /dev/centos/root
  Size of logical volume centos/root changed from <26.00 GiB (6655 extents) to 125.99 GiB (32254 extents).
  Logical volume centos/root successfully resized.

## 完成后，使用`df -Th`,仍未加上

## 还需要在系统，通过系统命令resyze2fs
[root@localhost xidian]# resize2fs /dev/centos/root 
resize2fs 1.42.9 (28-Dec-2013)
resize2fs: Bad magic number in super-block while trying to open /dev/centos/root
Couldn't find valid filesystem superblock.
# 这里可能报下面的错误但是可以解决，方法如下
```

- 如果报上面`Couldn't find valid filesystem superblock.`错误，可以[参考解决方法](http://www.systemadmin.me.uk/?p=434),使用下面命令：
```bash
[root@localhost xidian]# xfs_growfs /dev/mapper/centos-root 键入命令
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=1703680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=6814720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=3327, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 6814720 to 33028096
```
- 这个时候已经扩容完成`df -h`查看

#### 虚拟机中文件系统为直连时
```bash
fdisk /dev/vda
p  # 确定需要扩容的硬盘，以及对应的磁盘标签t。 此处为primary为2的。
d #删除vda2,
n #新建vda2，是否需要remove 标识符：no
no
t #查看文件系统标识符，是否正确。
w # 写入系统

sync
resize2fs /dev/vda # 使硬盘更新，同步到系统。
df -Th # 查看更新后的系统存储容量。
```
