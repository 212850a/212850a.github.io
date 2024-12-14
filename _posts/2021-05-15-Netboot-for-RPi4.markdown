---
layout: post
title: "Netboot for Raspberry Pi with /boot and root (/) on NFS"
date: 2021-05-15 09:00:00 +0200
tags: pxe netboot raspberrypi rpi nfs
published: true
---
## Preface
I started to think about netboot (pxe booting) for raspberry pi 4 some time ago. For the beginning it was just due to curiosity, but when I switched my k3s cluster to [HA mode](/2021/04/23/K3s-ha-kube-vip.html) and saw how much utilized disks on master nodes I've come to decision to test it for my cluster. 

After checking [official documentation](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md) I've found [really comprehensive guide](https://www.definit.co.uk/2020/03/pxe-booting-raspberry-pi-4-with-synology-and-ubiquiti-edgerouter/) about how to implement exactly what was needed in my environment. 
This post is almost identical copy of that guide with few important moments which were missed there.

## Plan
- Configure the DHCP service on your router
- Configure the NFS and TFTP services on your NAS
- Configure Raspberry Pi to store /boot and / (root) on NFS server 

## DHCP service
In order for the Raspberry Pi to know where to boot from, *tftp-server-name* option should be added into DHCP service. If Ubiquity router is used and 192.168.8.99 is ip-address of NFS/TFTP-service then it will look like:

```
set service dhcp-server shared-network-name LAN subnet 192.168.8.0/24 tftp-server-name 192.168.8.99
``` 

## NFS/TFTP service
On NAS the following two folders should be created:
* rpi-tftpboot - where /boot file systems for each RPi will be stored
* rpi-pxe - where root file systems for each RPi will be stored

NFS-service has to be active and RW-access should be provided for RPi-nodes to rpi-tftpboot and rpi-pxe folders. 

TFTP-service has to be enabled either and rpi-tftpboot folder should be specified as TFTP root folder.

## Raspberry Pi Boot
To continue further Raspbian OS should be installed into microSD as usually and desired configuration should be set:
- enable SSH-service
- generate locale
- setup time zone
- update system with sudo apt-get update && apt-get upgrade

Last step should be confirming serial number of RPi - it will be needed for next steps:
```
$ vcgencmd otp_dump | grep 28: | sed s/.*://g
47102626
```
The same microSD card can be used on different RPi's to confirm & note serials.

### Configure EEPROM
The Raspberry Pi 4 has an SPI-attached [EEPROM](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md) (4MBits/512KB), which contains code to boot up the system and replaces bootcode.bin previously found in the boot partition of the SD card. Note that if a bootcode.bin is present in the boot partition of the SD card in a Pi 4, it is ignored.
To go further EEPROM has to be updated - boot order has to be amended - boot from network should be added there as first step.

```
sudo apt install rpi-eeprom

# check what stable EEPROM binaries you have
ls -l /lib/firmware/raspberrypi/bootloader/stable
total 5424
-rw-r--r-- 1 root root 524288 Apr 23  2020 pieeprom-2020-04-16.bin
-rw-r--r-- 1 root root 524288 Jun 17  2020 pieeprom-2020-06-15.bin
-rw-r--r-- 1 root root 524288 Jul 20  2020 pieeprom-2020-07-16.bin
-rw-r--r-- 1 root root 524288 Aug 10  2020 pieeprom-2020-07-31.bin
-rw-r--r-- 1 root root 524288 Sep  7  2020 pieeprom-2020-09-03.bin
-rw-r--r-- 1 root root 524288 Dec 15 10:51 pieeprom-2020-12-11.bin
-rw-r--r-- 1 root root 524288 Jan 14 14:47 pieeprom-2021-01-11.bin
-rw-r--r-- 1 root root 524288 Jan 16 18:26 pieeprom-2021-01-16.bin
-rw-r--r-- 1 root root 524288 Feb 22 19:53 pieeprom-2021-02-16.bin
-rw-r--r-- 1 root root 524288 Mar 18 19:01 pieeprom-2021-03-18.bin
-rw-r--r-- 1 root root 106432 Apr 15 14:27 recovery.bin
-rw-r--r-- 1 root root  98904 Feb 28  2020 vl805-000137ad.bin
-rw-r--r-- 1 root root  99224 Jul 20  2020 vl805-000138a1.bin

# Copy the latest EEPROM binary to your homedir
cd ~
cp /lib/firmware/raspberrypi/bootloader/stable/pieeprom-2021-03-18.bin pieeprom.bin

# Export the default settings to a file
rpi-eeprom-config pieeprom.bin > bootconf.txt

# Edit the settings file and change the BOOT_ORDER to 0x21
vim bootconf.txt
[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0
BOOT_ORDER=0xf21

# Create a new EEPROM binary with the settings embedded
rpi-eeprom-config --out pieeprom-new.bin --config bootconf.txt pieeprom.bin

# Install the new EEPROM binary
sudo rpi-eeprom-update -d -f ./pieeprom-new.bin

# Reboot
sudo reboot
```
After reboot you can run rpi-eeprom-config command and you should get updated BOOT_ORDER.
```
rpi-eeprom-config
[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0
BOOT_ORDER=0xf21
```

### Copy Root to NFS server
Once the RPi has rebooted data of the root partition should be copied to the NFS server. As NFS-service is already configured and has rpi-pxe folder it can be done the following way:
```
# rsync will be used to copy data
sudo apt-get install rsync

sudo mount 192.168.8.99:/volume1/rpi-pxe /mnt

# Each RPi has to have its own root on NFS, p1 folder will be for first one
sudo mkdir /mnt/p1

# Copy the root OS partition to the NFS share, excluding the mnt mount (it will take a while)
sudo rsync -xa --progress --exclude /mnt / /mnt/p1
```
Once the copy has completed edit the /etc/fstab in the NFS share (so /mnt/p1/etc/fstab in example) - only /proc and /boot should be left and /boot should be specified as subfolder from 192.168.8.99:/volume1/rpi-tftpboot. The name of that subfolder should be exactly the same as serial of RPi (47102626 in example below):

```
sudo cat /mnt/p1/etc/fstab
proc            /proc           proc    defaults          0       0
192.168.8.99:/volume1/rpi-tftpboot/47102626 /boot nfs defaults,vers=3,proto=tcp 0 0
```

### Copy Boot to NFS server
```
# unmount /mnt if it was mounted before
sudo umount /mnt

# mount rpi-tftpboot to copy files from /boot there
sudo mount 192.168.8.99:/volume1/rpi-tftpboot /mnt

# Each RPi has to have its own boot on NFS
sudo mkdir /mnt/47102626

# 
sudo cp -r /boot/* /mnt/47102626

```
Next edit the cmdline.txt file in the new TFTP boot directory and replace the root partition with the NFS partition (folder where data was copied with rsync before).
```
sudo vim /mnt/47102626/cmdline.txt
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=192.168.8.99:/volume1/rpi-pxe/p1,vers=3 rw ip=dhcp elevator=deadline rootwait cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

### RPi 5
On RPi 5 (Raspbian OS 12 bookworm) to confirm serial number of RPi:

```
# cat /proc/cpuinfo | grep Serial | awk -F ': ' '{print $2}' | tail -c 9
92003445
```

cmdline.txt with all required files are moved under /boot/firmware, so only content of /boot/firmware has to be copied to rpi-tftpboot.

```
sudo mount 192.168.8.99:/volume1/rpi-tftpboot /mnt
sudo mkdir /mnt/92003445
sudo cp -r /boot/firmware/* /mnt/92003445
```
/etc/fstab file has to be like:
```
sudo mount 192.168.8.99:/volume1/rpi-pxe /mnt
cat /mnt/p1/etc/fstab
proc            /proc           proc    defaults          0       0
192.168.8.99:/volume1/rpi-tftpboot/92003445 /boot/firmware nfs defaults,vers=3,proto=tcp 0 0
```

### Boot RPi from network
Reboot your RPi and after couple of minutes it should be booted from network.
```
pi@p1:~ $ mount
192.168.8.99:/volume1/rpi-pxe/p1 on / type nfs (rw,relatime,vers=3,rsize=4096,wsize=4096,namlen=255,hard,nolock,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.8.99,mountvers=3,mountproto=tcp,local_lock=all,addr=192.168.8.99)
devtmpfs on /dev type devtmpfs (rw,relatime,size=1827468k,nr_inodes=97607,mode=755)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,relatime)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
none on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=37,pgrp=1,timeout=0,minproto=5,maxproto=5,direct)
mqueue on /dev/mqueue type mqueue (rw,relatime)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
sunrpc on /run/rpc_pipefs type rpc_pipefs (rw,relatime)
configfs on /sys/kernel/config type configfs (rw,relatime)
192.168.8.99:/volume1/rpi-tftpboot/47102626 on /boot type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.8.99,mountvers=3,mountport=892,mountproto=tcp,local_lock=none,addr=192.168.8.99)
```

## K3s on RPi with root and /boot on NFS
There is important moment which you need to remember - if you plan to run K3s on RPi with netboot - overlayfs which is used as default for containerd does not support NFS, as result K3s has to be started with '--snapshotter=native' option.
Example for inventory/group_vars/all.yml file from [K3s in HA mode with kube-vip](/2021/04/23/K3s-ha-kube-vip.html)
```
cat inventory/group_vars/all.yml
---
k3s_version: v1.19.8+k3s1
ansible_user: vagrant
systemd_dir: /etc/systemd/system
flannel_iface: "eth1"
apiserver_endpoint: "192.16.35.100"
k3s_token: "mysupersecuretoken"
{% raw  %}extra_server_args: "--node-ip={{ ansible_eth1.ipv4.address }} --flannel-iface={{ flannel_iface }} --no-deploy servicelb --no-deploy traefik --snapshotter=native"
extra_agent_args: "--flannel-iface={{ flannel_iface }} --snapshotter=native"{% endraw %}
```

## Links
* [Network boot your Raspberry Pi](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md)
* [PXE Booting Raspberry Pi 4 with Synology and Ubiquiti EdgeRouter](https://www.definit.co.uk/2020/03/pxe-booting-raspberry-pi-4-with-synology-and-ubiquiti-edgerouter/)
* [K3S NFS Root howto](http://chegwin.org/posts/k3s_raspi_nfs_mount/)
