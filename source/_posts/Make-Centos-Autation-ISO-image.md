---
title: Make Centos Autation ISO image
date: 2022-03-10 12:16:05
tags: Tech
categories: Technology
---

本讨论记录Autation Centos7制作过程。功能为无人值守安装自定义镜像。

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

UEFI部分：

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

BIOS部分：

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
redhat-lsb-core
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

BIOS部分：

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

A7. 启动菜单添加自定义kickstart启动

cd Autation/boot-menu

UEFI部分：

```shell
[root@analysis boot-menu]# cat grub.cfg.template 
set default="0"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=60
### END /etc/grub.d/00_header ###

search --no-floppy --set=root -l 'CentOS 7 x86_64'

### BEGIN /etc/grub.d/10_linux ###
menuentry 'Install CentOS 7 - Autation Release' --class fedora --class gnu-linux --class gnu --class os {
        linuxefi /images/pxeboot/vmlinuz inst.ks=hd:LABEL=CentOS\x207\x20x86_64:/Autation/ks-cfg/uefi-ks.cfg 
        inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet
        initrdefi /images/pxeboot/initrd.img
}
menuentry 'Test this media & install CentOS 7' --class fedora --class gnu-linux --class gnu --class os {
        linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet
        initrdefi /images/pxeboot/initrd.img
}
submenu 'Troubleshooting -->' {
        menuentry 'Install CentOS 7 in basic graphics mode' --class fedora --class gnu-linux --class gnu --class os {
                linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 xdriver=vesa nomodeset quiet
                initrdefi /images/pxeboot/initrd.img
        }
        menuentry 'Rescue a CentOS system' --class fedora --class gnu-linux --class gnu --class os {
                linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rescue quiet
                initrdefi /images/pxeboot/initrd.img
        }
}
```

BISO部分：

```shell
[root@analysis boot-menu]# cat isolinux.cfg.template 
default vesamenu.c32
timeout 600

display boot.msg

# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu background splash.png
menu title CentOS 7
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none

# Selected item
menu color sel 0 #ffffffff #00000000 none

# Title bar
menu color title 0 #ff7ba3d0 #00000000 none

# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none

# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none

# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none

# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none

# Help text
menu color help 0 #ffffffff #00000000 none

# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none

# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install CentOS 7 - Autation Release
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.ks=hd:LABEL=CentOS\x207\x20x86_64:/Autation/ks-cfg/bios-ks.cfg inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet

label check
  menu label Test this ^media & install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet

menu separator # insert an empty line

# utilities submenu
menu begin ^Troubleshooting
  menu title Troubleshooting

label vesa
  menu indent count 5
  menu label Install CentOS 7 in ^basic graphics mode
  text help
        Try this option out if you're having trouble installing
        CentOS 7.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 xdriver=vesa nomodeset quiet

label rescue
  menu indent count 5
  menu label ^Rescue a CentOS system
  text help
        If the system will not boot, this lets you access files
        and edit config files to try to get it booting again.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rescue quiet

label memtest
  menu label Run a ^memory test
  text help
        If your system is having issues, a problem with your
        system's memory may be the cause. Use this utility to
        see if the memory is working correctly.
  endtext
  kernel memtest

menu separator # insert an empty line

label local
  menu label Boot from ^local drive
  localboot 0xffff

menu separator # insert an empty line
menu separator # insert an empty line

label returntomain
  menu label Return to ^main menu
  menu exit

menu end
```



A8. 生成iso镜像文件
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

