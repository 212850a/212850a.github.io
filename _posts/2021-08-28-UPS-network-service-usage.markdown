---
layout: post
title: "UPS network service usage"
date: 2021-08-28 09:00:00 +0200
tags: UPS cluster network electricity
published: true
---
## Preface
It's quite risky to lose main power for Kubernetes cluster and it's even more risky if all drives on nodes are provided via [iSCSI](/2021/06/13/Diskless-RPi-over-iSCSI.html). In my setup UPS is connected to Synology NAS via USB cable, so Synology NAS knows when something is wrong with main power and can shutdown itself gracefully, however what to do with RPi cluster which is running on iSCSI volumes? Answer is simple - Network UPS Tools (NUT).

## Server side
Synology NAS has ability to enable UPS server (to work as master), in addition to that all RPi nodes should be specifiedin "Permitted DiskStation devices".
![psynology-ups](/assets/synology-ups.png)

## Client side
On all RPi's NUT packet has to be installed and configured in client mode (slave mode).
```
apt-get install nut -y

vi /etc/nut/nut.conf
MODE=netclient

vi /etc/nut/upsmon.conf
MONITOR ups@192.168.8.99 1 monuser secret slave

service nut-client restart
service nut-client status
● nut-monitor.service - Network UPS Tools - power device monitor and shutdown controller
   Loaded: loaded (/lib/systemd/system/nut-monitor.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-07-19 09:59:28 EEST; 57s ago
  Process: 3961 ExecStart=/sbin/upsmon (code=exited, status=0/SUCCESS)
 Main PID: 3963 (upsmon)
    Tasks: 2 (limit: 4915)
   Memory: 1.0M
   CGroup: /system.slice/nut-monitor.service
           ├─3962 /lib/nut/upsmon
           └─3963 /lib/nut/upsmon

```
With a help of upsc command it is possible to confirm if UPS server returns required data about UPS status. The main values are battery.charge (%), battery.charge.low (%), battery.runtime (sec), battery.runtime.low (sec) and ups.status (OL - online):
```
upsc ups@192.168.8.99
Init SSL without certificate database
battery.charge: 100
battery.charge.low: 10
battery.charge.warning: 50
battery.date: 2021/08/15
battery.mfr.date: 2021/08/15
battery.runtime: 3007
battery.runtime.low: 120
battery.temperature: 29.2
battery.type: PbAc
battery.voltage: 13.5
battery.voltage.nominal: 12.0
device.mfr: American Power Conversion
device.model: Back-UPS CS 350
device.serial: 4B0841P06334
device.type: ups
driver.name: usbhid-ups
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 5
driver.parameter.port: auto
driver.version: SDS6-0-8700-factory-repack-8700-170413
driver.version.data: APC HID 0.95
driver.version.internal: 0.38
input.sensitivity: low
input.transfer.high: 278
input.transfer.low: 160
input.voltage: 234.0
input.voltage.nominal: 230
output.frequency: 50.0
output.voltage: 230.0
output.voltage.nominal: 230.0
ups.beeper.status: enabled
ups.delay.shutdown: 20
ups.delay.start: 30
ups.firmware: 807.q8.I
ups.firmware.aux: q8
ups.load: 16.0
ups.mfr: American Power Conversion
ups.mfr.date: 2008/10/07
ups.model: Back-UPS CS 350
ups.productid: 0002
ups.realpower.nominal: 210
ups.serial: 4B0841P06334
ups.status: OL
ups.test.result: No test initiated
ups.timer.reboot: 0
ups.timer.shutdown: -1
ups.timer.start: 0
ups.vendorid: 051d
```
## How it works
When a UPS goes critical (status on battery + reaches low battery, or "FSD": forced shutdown), the slaves are supposed to disconnect and shut down right away. After all slaves went to shutdown master goes down either. 

To test shutdown FSD can be initiated on slave (in that case only that slave goes down) or master (in that case all slaves will go down) with a help of command:
```
upsmon -c fsd
```
In the logs it will be see like:
```
Aug 28 13:52:21 p1 upsmon[599]: UPS ups@192.168.8.99: forced shutdown in progress
Aug 28 13:52:21 p1 upsmon[599]: Executing automatic power-fail shutdown
Aug 28 13:52:21 p1 upsmon[599]: Auto logout and shutdown proceeding
```
## My own moments
- For Back-UPS CS 350 after old battery replacement status of battery still returned as OL LO (online low) with runtime less than low level. I've found only one way how to fix it - install PowerChute Personal Edition on Windows machine with connected UPS and then push "Replace Battery Date" and "Run Self-Test". See [How to change the battery replacement date of a Back-UPS using PowerChute Personal Edition?](https://www.se.com/ww/en/faqs/FA408884/) for more details.