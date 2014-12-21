---
layout: post
title: 虚拟机磁盘损坏启动失败故障处理
tags: 
keywords: kvm, fsck, libvirtd
description: 虚拟机磁盘损坏了，即不能启动执行fsck，也没有独立硬盘，该如何是好
---

使用的是 kvm 虚拟化，Libvirtd管理，所有虚拟机都是clone自某台模板机，但前不久对 母机 进行一次大文件 拷贝操作（从 NTFS到Ext4）时，按了Ctrl-C，导致母机异常Kernel Panic（后来得知是母机配置的RAID0中的某一块硬盘故障导致），母机随后被强制关机，从而导致所有的虚拟机被强制关机。

后经过若干方法重新使得母机RAID0恢复，最终使得母机正常开机，而后检查各虚拟机，大部分正常启动，便未继续予以关注。但后来同事爆出其中最重要的数据库机器SSH连不上且无法ping通，此机器上不仅有数据库、还有部门内部svn等等。

使用virt-manager，重启此虚拟机，观察其控制台输出，发现在其initramfs阶段，挂载rootfs出错，导致kernel pannic。出现这种情况一般就是硬盘故障了，启动后使用fsck.ext4（参考了[这里][1]）即可恢复问题分区，若无法开机则将其硬盘拆下，挂载至其他Linux机器亦可。但奇葩的是，两个比较棘手的问题同时出现了，一则无法开机，二则此机器乃虚拟机，并无独立的硬盘，实际上机器的所有存储就只有一个独立的img文件，又该如何将其拆下呢。忘了Loop设备吧，Qemu的磁盘镜像文件并不是Loop设备，不能挂载为字符设备的，更何况若使用了其他压缩格式，就更不可能是Loop设备了。

数据找不回了么，明显真实数据仍摆在那，而且必须找回。先对其进行备份，以防误操作：

```bash
$ cp tpl-slave4.img slave4_bak.img

cp reading tpl-slave4.img Input/Output Error
```

果不其然，复制错误，检查slave4_bak.img文件，发现只复制了接近2.5G的文件，虽然复合此盘上的实际文件预期，但始终不放心。

还是按照方法二，将其硬盘拆下，挂入其他机器：

```bash
$ virsh start tpl-slave1

$ virsh attach-disk tpl-slave1 vdb /home/ue/tpl-slave4.img
```

启动另一台虚拟机 tpl-slave1，并将源盘挂载至新机器，执行`lvscan`，系统提示：`Found duplicate PV` ，原来，所有的机器均克隆自同一台机器，lvm卷管理器无法区分应该使用哪一个底层PV（[这里][2]的办法可以选择性使用哪一个PV，但我当时选择了另外的方法）。遂新建一虚拟机并启动CentOS LiveCD，并将损坏的img文件做为Live机器的磁盘二装载，执行：

`vgchange -ay /dev/vg_tpl`

激活lvm卷

挂载目标分区，然后fsck.ext4，完成后直接关机，删除新建的LiveCD机器，尝试启动原有机器，成功。

  [1]: http://www.cnblogs.com/dancefire/archive/2011/03/09/fix-bad-superblock-in-linux.html
  [2]: http://blog.slogra.com/post-437.html
