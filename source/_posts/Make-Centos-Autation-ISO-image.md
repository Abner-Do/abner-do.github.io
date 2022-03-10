---
title: Make Centos Autation ISO image
date: 2022-03-10 12:16:05
tags: Tech
categories: Technology
---

本讨论记录Autation Centos7制作过程。功能为自定义镜像自动安装。

A1.环境准备
Platform:  Centos7

Requires: 安装操作系统硬盘空间大于100G。

A2.拷贝centos7 Minimal 镜像文件

```shell
[root@analysis home]# mount CentOS-7-x86_64-Minimal-2009.iso /media/
mount: /dev/loop0 is write-protected, mounting read-only
[root@analysis home]# mkdir -p /home/CTGMAS-PGMAS-CentOS7
[root@analysis home]# rsync -avzP /media/ /home//CentOS7-Autation
```
/home/CentOS7-Autation 目录结构：

```shell
[root@analysis home]# tree -L 1 /home/CentOS7-Autation
/home/CentOS7-Autation
├── Autation
├── CentOS_BuildTag
├── EFI
├── EULA
├── GPL
├── images
├── isolinux
├── LiveOS
├── Packages
├── repodata
├── RPM-GPG-KEY-CentOS-7
├── RPM-GPG-KEY-CentOS-Testing-7
└── TRANS.TBL

7 directories, 6 files
```

A3.硬盘分区

```shell
cd /home/CentOS7-Autation
mkdir -p Autation/kickstart-scripts/
vim Autation/kickstart-scripts/uefi-partdrive.sh
[root@analysis kickstart-scripts]# cat uefi-partdrive.sh 
#!/bin/bash
LANG=C

for file in /sys/block/*da;do
  diskname="$(basename $file)"
  if [[ "sda" == "$diskname" ]]; then
     diskname="sda"
     break;
  elif [[ "hda" == "$diskname" ]]; then
     diskname="hda"
     break;
  elif [[ "xvda" == "$diskname" ]]; then
     diskname="xvda"
     break;
  elif [[ "vda" == "$diskname" ]]; then
     diskname="vda"
     break;
  fi
done
echo "diskname: $diskname"

cap=$(gdisk  -l /dev/$diskname|grep "Disk /dev/$diskname"|awk -F ','  '{print $2}'|awk -F ' '  '{print $1}')
unit=$(gdisk -l /dev/$diskname|grep "Disk /dev/$diskname"|awk -F ','  '{print $2}'|awk -F ' '  '{print $2}')
echo "  cap: $cap  unit: $unit"

lastUsableSector=$(gdisk -l /dev/$diskname |grep "last usable sector"|awk -F ',' '{print $2}'|awk -F ' ' '{print $5}')
#Available sectors.
totalSector=$(($lastUsableSector-2048+1))
#totalSpace, the unit is MiB.
totalCap=$(($totalSector*512/1024/1024))

echo "totalCap: $totalCap M"

bootCap=500
efiCap=200
remainCap=$(($totalCap-$bootCap-$efiCap))
#rootCap=$(($remainCap*2/3))
rootCap=$((100*1024))
remainCap=$(($remainCap-$rootCap))
homeCap=$remainCap

echo "boot: $bootCap efi: $efiCap root: $rootCap "

echo "clearpart --all --initlabel --disklabel gpt --drives $diskname" > /tmp/part-include
echo "ignoredisk --only-use $diskname" >> /tmp/part-include
echo "part /boot --fstype xfs --size $bootCap " >> /tmp/part-include
echo "part /boot/efi --fstype=efi --size $efiCap --fsoptions=\"umask=0077,shortname=winnt\" " >> /tmp/part-include
echo "part pv.01 --fstype lvmpv --size $(($totalCap-200-500))  --maxsize $(($totalCap-200-500)) --ondisk $diskname --grow" >> /tmp/part-include
echo "volgroup centos --pesize 4096 pv.01" >> /tmp/part-include
#echo "logvol swap  --fstype swap --hibernation --name lv_swap --vgname centos" >> /tmp/part-include
echo "logvol swap  --fstype swap --size 16384 --name lv_swap --vgname centos" >> /tmp/part-include
echo "logvol /     --fstype xfs --size $rootCap --grow --name lv_root --vgname centos" >> /tmp/part-include
#echo "logvol /     --fstype xfs --size $rootCap --maxsize $rootCap --name lv_root --vgname centos" >> /tmp/part-include
#echo "logvol /home --fstype xfs --size 1 --grow --name lv_home --vgname centos" >> /tmp/part-include
```

A4.准备需要的安装包
 安装包列表
 vim Autation/packages-list

```shell
vim
bash-completion
tmux
rsync
gcc
gcc-c++
kernel-devel
samba4
yum-utils
docker-ce
docker-ce-cli
containerd.io
lrzsz
strace
lsb_release
telnet
mydumper
```

存放安装包的目录

```shell
mkdir -p Autation/Packages/Extra
mkdir -p Autation/Packages/Updates
```

其中Extra目录存放自定义安装包，Updates存放更新包，可以使用yum下载软件包，不安装。

```shell
yum --downloadonly --downloaddir /opt/Extra install <package_name>
yum --downloadonly --downloaddir /opt/Updates update 
```

生成repodata

```shell
createrepo Autation/Packages/Extra
createrepo Autation/Packages/Updates
```

A5.密码生成

```shell
#!/bin/python3
# -*- coding:utf-8 -*-

import crypt
import random,string

print (crypt.crypt('YuE1Z7p%',"$6$YuE1Z7p%"))
```

A6.制作kickstart配置文件，文件生成模板可以手动安装在/root目录发现anaconda-ks.cfg，或者用软件生成。
vim /home/CentOS7-Autation/Autation/ks-cfg/uefi-ks.cfg

```shell
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
cdrom
# Use text install
text
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=em1 --onboot=off --ipv6=auto --no-activate
network  --hostname=localhost.localdomain

# Root password
rootpw --iscrypted $6$YuE1Z7p%$LtwmybWumyUliwpcyElyE8S00w6DE3/j21.8tobeG5iZHwym33ccccppdMP7.JsgIpBWefG38zfprrgmh.Zg00
# System services
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc
#Creat an administrator
user --groups=wheel --name=bradmin --password=$6$YuE1Z7p%$LtwmybWumyUliwpcyElyE8S00w6DE3/j21.8tobeG5iZHwym33ccccppdMP7.JsgIpBWefG38zfprrgmh.Zg00 iscrypted --gecos="BRadmin"
selinux --disabled
#After installation complete
shutdown

%pre --log=/root/autapre.log
sh /run/install/repo/Autation/kickstart-scripts/uefi-partdrive.sh
%end
%include /tmp/part-include

repo --name=ExtraPackages --baseurl=file:///run/install/repo/Autation/Packages/Extra
repo --name=updates --baseurl=file:///run/install/repo/Autation/Packages/Updates

%packages 
@^minimal
@core
chrony
kexec-tools
%include /run/install/repo/Autation/Packages/packages-list

%end

%post 
systemctl enable docker
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

A7.生成iso镜像文件
 制作脚本

```shell
[root@analysis Autation]# cat mkiso.sh 
#!/bin/bash

#Set Volume ID 
VolumeID="CentOS 7 x86_64"
#Set Output Image Name
OutputImageName="../../CentOS7-Autation-R$(date +%y.%m.%d).iso"
#Set Source Image directory to generate New Image
ImageDirectory=".."
sed 's/Release/R'`date +%y.%m.%d`'/' ./boot-menu/grub.cfg.template > ../EFI/BOOT/grub.cfg
sed 's/Release/R'`date +%y.%m.%d`'/' ./boot-menu/isolinux.cfg.template > ../isolinux/isolinux.cfg

xorriso -as mkisofs \
  -c isolinux/boot.cat \
  -b isolinux/isolinux.bin \
  -no-emul-boot \
  -boot-load-size 4 \
  -boot-info-table \
  -eltorito-alt-boot \
  -e images/efiboot.img \
  -isohybrid-gpt-basdat \
  -no-emul-boot \
  -o $OutputImageName \
  -V "$VolumeID" \
  -R -J  $ImageDirectory

implantisomd5 $OutputImageName
```
 执行命令生成iso生成镜像 
```
bash  mkiso.sh
```

查看生成的镜像

```shell
[root@analysis home]# ll -d /home/CentOS7-Autation-R22.02.11.iso 
-rw-r--r-- 1 root root 1588158464 Feb 11 09:46 /home/CentOS7-Autation-R22.02.11.iso
```

添加bios启动
 B1. kickstart配置文件

```shell
[root@analysis Autation]# cat bios-ks.cfg 
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
cdrom
# Use text install
text
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=em1 --onboot=off --ipv6=auto --no-activate
network  --hostname=localhost.localdomain

# Root password
rootpw --iscrypted $6$YuE1Z7p%$LtwmybWumyUliwpcyElyE8S00w6DE3/j21.8tobeG5iZHwym33ccccppdMP7.JsgIpBWefG38zfprrgmh.Zg00
# System services
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc
#Creat an administrator
user --groups=wheel --name=bradmin --password=$6$YuE1Z7p%$LtwmybWumyUliwpcyElyE8S00w6DE3/j21.8tobeG5iZHwym33ccccppdMP7.JsgIpBWefG38zfprrgmh.Zg00 iscrypted --gecos="BRadmin"
selinux --disabled
#After installation complete
shutdown

# System bootloader configuration
#bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# Partition clearing information
#clearpart --none --initlabel
# Disk partitioning information
#part pv.156 --fstype="lvmpv" --ondisk=sda --size=4193277
#part /boot --fstype="xfs" --ondisk=sda --size=1024
#part biosboot --fstype="biosboot" --ondisk=sda --size=1
#volgroup centos --pesize=4096 pv.156
#logvol /home  --fstype="xfs" --grow --size=500 --name=home --vgname=centos
#logvol swap  --fstype="swap" --size=3968 --name=swap --vgname=centos
#logvol /  --fstype="xfs" --grow --maxsize=51200 --size=1024 --name=root --vgname=centos

%pre --log=/root/autapre.log
sh /run/install/repo/Autation/kickstart-scripts/bios-partdrive.sh
%end
%include /tmp/part-include

repo --name=ExtraPackages --baseurl=file:///run/install/repo/Autation/Packages/Extra
repo --name=updates --baseurl=file:///run/install/repo/Autation/Packages/Updates

%packages
@^minimal
@core
chrony
kexec-tools
%include /run/install/repo/Autation/Packages/packages-list
%end

%post 
systemctl enable docker
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

B2.分区部分

```shell
[root@analysis kickstart-scripts]# cat bios-partdrive.sh 
#!/bin/bash

LANG=C

for file in /sys/block/*da;do
  diskname="$(basename $file)"
  if [[ "sda" == "$diskname" ]]; then
     diskname="sda"
     break;
  elif [[ "hda" == "$diskname" ]]; then
     diskname="hda"
     break;
  elif [[ "xvda" == "$diskname" ]]; then
     diskname="xvda"
     break;
  elif [[ "vda" == "$diskname" ]]; then
     diskname="vda"
     break;
  fi
done
echo "diskname: $diskname"

cap=$(gdisk  -l /dev/$diskname|grep "Disk /dev/$diskname"|awk -F ','  '{print $2}'|awk -F ' '  '{print $1}')
unit=$(gdisk -l /dev/$diskname|grep "Disk /dev/$diskname"|awk -F ','  '{print $2}'|awk -F ' '  '{print $2}')
echo "  cap: $cap  unit: $unit"

lastUsableSector=$(gdisk -l /dev/$diskname |grep "last usable sector"|awk -F ',' '{print $2}'|awk -F ' ' '{print $5}')
#Available sectors.
totalSector=$(($lastUsableSector-2048+1))
#totalSpace, the unit is MiB.
totalCap=$(($totalSector*512/1024/1024))

echo "totalCap: $totalCap M"

bootCap=500
remainCap=$(($totalCap-$bootCap-1))
#rootCap=$(($remainCap*2/3))
rootCap=$((100*1024))
remainCap=$(($remainCap-$rootCap))
homeCap=$remainCap

echo "boot: $bootCap root: $rootCap "

# System bootloader configuration
echo "bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=$diskname"> /tmp/part-include
# Partition clearing information
echo "clearpart --all --initlabel --disklabel gpt --drives $diskname" >> /tmp/part-include
# Disk partitioning information
echo "ignoredisk --only-use $diskname" >> /tmp/part-include
echo "part /boot --fstype xfs --size $bootCap " >> /tmp/part-include
echo "part biosboot --fstype="biosboot" --ondisk=sda --size=1" >> /tmp/part-include
echo "part pv.02 --fstype lvmpv --size $(($totalCap-500-1))  --maxsize $(($totalCap-500-1)) --ondisk $diskname --grow" >> /tmp/part-include
echo "volgroup centos --pesize 4096 pv.02" >> /tmp/part-include
#echo "logvol swap  -dd-fstype swap --hibernation --name lv_swap --vgname centos" >> /tmp/part-include
echo "logvol swap  --fstype swap --size 16384 --name lv_swap --vgname centos" >> /tmp/part-include
echo "logvol /     --fstype xfs --size $rootCap --grow --name lv_root --vgname centos" >> /tmp/part-include
#echo "logvol /     --fstype xfs --size $rootCap --maxsize $rootCap --name lv_root --vgname centos" >> /tmp/part-include
#echo "logvol /home --fstype xfs --size 1 --grow --name lv_home --vgname centos" >> /tmp/part-include
```
