---
layout: post
title: "Diskless Raspberry Pi over iSCSI"
date: 2021-06-13 09:00:00 +0200
tags: pxe netboot raspberrypi rpi iscsi
published: true
---
## Preface
Several weeks of testing [Netboot for Raspberry Pi with /boot and root (/) on NFS](/2021/05/15/Netboot-for-RPi4.html) went fine until I hit a [problem](https://github.com/k3s-io/k3s/issues/2312#issuecomment-851301070) with home-assistant due to container didn't want to start at all with strange message *failed to reserve container name is reserved for*. 

I knew that there is iSCSI feature on my Synology, but I never tried it before. So the time has come to try.

Plan was quite simple - partially to use what already was done with PXE and booting from TFTP, but now root (/) should be on iSCSI disks, not on NFS mount as before, /boot can be still left on NFS mount.

After reading several good articles (all of them are left at the end) I've understood that in addition to what was done initrd has to be recompiled with iscsi modules as by default they are not there so OS won't start without it from iscsi disk.


## Plan
So at the end I've got the following release plan:
- Configure the DHCP service on your router (the same as [Netboot for Raspberry Pi with /boot and root (/) on NFS](/2021/05/15/Netboot-for-RPi4.html))
- Configure the NFS and TFTP services on your NAS (only /boot will be on NFS now)
- Configure iSCSI and test it (on storage and RPi ends)
- Recompile initrd with iscsi modules on RPi
- Copy /boot to NFS and update boot configuration for iSCSI
- Copy / to iSCSI disk

## NFS/TFTP service
On NAS rpi-tftpboot folder should be created, /boot file systems for each RPi will be stored there.

NFS-service has to be active and RW-access should be provided for RPi-nodes to rpi-tftpboot folder. 

TFTP-service has to be enabled either and rpi-tftpboot folder should be specified as TFTP root folder.

## Configure iSCSI and test it
### Storage
From storage side volumes (disks/LUNs) have to be created and mapped to targets (one target per each RPi server), I use 16GB volumes (one for each RPi) and simple CHAP authentication (not mutual). On examples below 192.168.8.99 is my Synology.
### Server/RPi
From server side open-iscsi packet has to be installed and configured
```
apt-get install open-iscsi -y
```
In /etc/iscsi/iscsid.conf file the same authentication credentials should be specified RPi to be able to login to iSCSI server to see available LUN. Restart service after changing configuration.
```
# egrep -v "^#" /etc/iscsi/iscsid.conf | egrep "(node.session.auth)|node.startup"
node.startup = automatic
node.session.auth.authmethod = CHAP
node.session.auth.username = rpi1
node.session.auth.password = mysupersecurepassword

# systemctl restart iscsid.service
```
Now you can try to run discovery and see what is available on iscsi-server (storage)
```
iscsiadm -m discovery -t st -p 192.168.8.99
192.168.8.99:3260,1 iqn.2000-01.com.synology:storage.rpi1.ebab984136
```
Let's try to login:
```
# iscsiadm   --mode node  --targetname "iqn.2000-01.com.synology:storage.rpi1.ebab984136" --portal 192.168.8.99 --login
Logging in to [iface: default, target: iqn.2000-01.com.synology:storage.rpi1.ebab984136, portal: 192.168.8.99,3260] (multiple)
Login to [iface: default, target: iqn.2000-01.com.synology:storage.rpi1.ebab984136, portal: 192.168.8.99,3260] successful.
```
Now we should be able to see new disk available (/dev/sda) like usual local drive:
```
# lsblk 
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda    8:0    0  16G  0 disk 
```
It can be formatted as ext4, just remember UUID of new created FS - it will be needed configuring boot later.
```
mkfs.ext4 /dev/sda
mkdir /mnt/iscsi
mount /dev/sda /mnt/iscsi
blkid /dev/sda
/dev/sda: UUID="de208c76-4d29-467f-b6c5-4d7c9f0e0827" TYPE="ext4"
```
## Recompile initrd with iscsi modules on RPi
Pretty simple and fast procedure, just don't forget to update system before to do that.
```
apt-get update && apt-get upgrade -y

touch /etc/iscsi/iscsi.initramfs
update-initramfs -v -k `uname -r` -c

ls -l /boot/initrd.img-5.10.17-v7l+
-rwxrwxrwx 1 1024 users 8792648 Jun 12 12:31 /boot/initrd.img-5.10.17-v7l+
```

## Copy /boot to NFS and update boot configuration for iSCSI
The same way as it was mentioned in [Netboot for Raspberry Pi with /boot and root (/) on NFS](/2021/05/15/Netboot-for-RPi4.html) post /boot should be copied to NFS rpi-tftpboot folder, each RPi should have their own boot sub-folder there (based on RPi s/n).
```
vcgencmd otp_dump | grep 28: | sed s/.*://g
47102626

mkdir /mnt/boot
sudo mount 192.168.8.99:/volume1/rpi-tftpboot /mnt/boot
sudo cp -r /boot/* /mnt/boot/47102626
```
Add to config.txt details about what initrd should be used
```
cat >> /mnt/boot/47102626/config.txt
initramfs initrd.img-5.10.17-v7l+ followkernel
[Ctrl+D]
```
To update cmdline.txt the following details need to know, all of them should be specified instead of current details about root device:
- root=UUID=de208c76-4d29-467f-b6c5-4d7c9f0e0827
- ISCSI_USERNAME=rpi1
- ISCSI_PASSWORD=mysupersecurepassword
- ISCSI_INITIATOR=iqn.1993-08.org.debian:01:3f3ea6e86ba8 (can be taken from /etc/iscsi/initiatorname.iscsi)
- ISCSI_TARGET_NAME=iqn.2000-01.com.synology:storage.rpi1.ebab984136
- ISCSI_TARGET_IP=192.168.8.99
- ISCSI_TARGET_PORT=3260
- rootfstype=ext4

So cmdline.txt should look like that:
```
cat /mnt/boot/47102626/cmdline.txt
console=serial0,115200 console=tty1 ip=dhcp root=UUID=de208c76-4d29-467f-b6c5-4d7c9f0e0827 ISCSI_USERNAME=rpi1 ISCSI_PASSWORD=mysupersecurepassword ISCSI_INITIATOR=iqn.1993-08.org.debian:01:d52b48145cc ISCSI_TARGET_NAME=iqn.2000-01.com.synology:storage.rpi1.ebab984136 ISCSI_TARGET_IP=192.168.8.99 ISCSI_TARGET_PORT=3260 rw rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

## RPi 5
On RPi 5 (Raspbian OS 12 bookworm) cmdline.txt and config.txt are in /boot/firmware. 
Generated initrd-file has to be copied to /boot/firmware either.

## Copy / to iSCSI disk
Last step is to copy all data from current / to iscsi disk which is still mounted as /mnt/iscsi
```
rsync -xa --progress --exclude /mnt / /mnt/iscsi
```
And edit copied /etc/fstab to mount /boot from NFS and specify new /
```
cat > /mnt/iscsi/etc/fstab
proc            /proc           proc    defaults          0       0
192.168.8.99:/volume1/rpi-tftpboot/47102626 /boot nfs defaults,vers=3,proto=tcp 0 0
UUID=de208c76-4d29-467f-b6c5-4d7c9f0e0827	/               ext4    defaults  1       1
```

## Reboot
After all these steps run reboot and check how it goes. System should boot via PXE, login to iSCSI server, get root disk visible and start from it as usually. After reboot you'll see something like:
```
# df -h
Filesystem                                 	Size  Used Avail Use% Mounted on
udev                                       	1.8G     0  1.8G   0% /dev
tmpfs                                      	383M   17M  367M   5% /run
/dev/sda                                   	16G  5.0G  9.9G  34% /
tmpfs                                    	1.9G     0  1.9G   0% /dev/shm
tmpfs                                      	5.0M  4.0K  5.0M   1% /run/lock
tmpfs                                      	1.9G     0  1.9G   0% /sys/fs/cgroup
192.168.8.99:/volume1/rpi-tftpboot/47102626	5.5T  4.2T  1.3T  77% /boot
```

And now home-assistant container works as expected in K3s.

## Links
* [Net boot (PXE + iSCSI) with a RaspberryPi 3](https://stuff.drkn.ninja/post/2016/11/08/Net-boot-(PXE-iSCSI)-with-a-RaspberryPi-3)
* [Raspberry Pi iSCSI Root on Ubuntu 20.04](https://matt.olan.me/raspberry-pi-iscsi-root-on-ubuntu-20-04/)
* [Network Booting a Raspberry Pi 4 with an iSCSI Root via FreeNAS](https://shawnwilsher.com/2020/05/network-booting-a-raspberry-pi-4-with-an-iscsi-root-via-freenas/)
* [Diskless Boot for a Raspberry Pi over PXE and iSCSI](https://tech.xlab.si/blog/pxe-boot-raspberry-pi-iscsi/)
