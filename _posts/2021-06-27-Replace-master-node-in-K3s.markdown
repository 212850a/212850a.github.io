---
layout: post
title: "How to replace master node in K3s"
date: 2021-06-27 09:00:00 +0200
tags: k3s master kubernetes
published: true
---
## Preface
Last week due to storm there was power outage in my apartment so storage went down, as result my K3s cluster went down either (as I'm using [Diskless Raspberry Pi over iSCSI](/2021/06/13/Diskless-RPi-over-iSCSI.html)). After power was restored and storage came online K3s on one of three masters didn't want to start. In /varlog/syslog I got the following:
```
Jun 25 10:04:21 p1 k3s[1389]: panic: freepages: failed to get all reachable pages (page 846222707: out of bounds: 5568)
Jun 25 10:04:21 p1 k3s[1389]: goroutine 213 [running]:
Jun 25 10:04:21 p1 k3s[1389]: github.com/rancher/k3s/vendor/go.etcd.io/bbolt.(*DB).freepages.func2(0x75dce00)
Jun 25 10:04:21 p1 k3s[1389]: #011/go/src/github.com/rancher/k3s/vendor/go.etcd.io/bbolt/db.go:1003 +0xcc
Jun 25 10:04:21 p1 k3s[1389]: created by github.com/rancher/k3s/vendor/go.etcd.io/bbolt.(*DB).freepages
Jun 25 10:04:21 p1 k3s[1389]: #011/go/src/github.com/rancher/k3s/vendor/go.etcd.io/bbolt/db.go:1001 +0x10c
```
After googling I've understood that etcd data on p1 master was corrupted and the quickest way would be just to remove & add it back to cluster.

## Plan
- Remove problematic master node from k3s cluster
- Prepare problematic master node to join to cluster
- Join master node to cluster

## Log
### Remove problematic master node from k3s cluster
First of all problematic master node has to be removed from cluster:
```
kubectl delete node p1
```
### Prepare problematic master node to join to cluster
On problematic node k3s service should be stopped as it tries to restart forever:
```
root@p1:~# systemctl stop k3s
```

Move problematic etcd database to separate folder
```
cd /var/lib/rancher/k3s/server
mv db db.old
```

Check and confirm there is no non-standard manifests are stored in /var/lib/rancher/k3s/server/manifests/ on new master node, otherwise it will be applied during k3s start and it may prevent joining to the cluster. I had to remove /var/lib/rancher/k3s/server/manifests/vip.yaml due to I use [K3s in HA mode with kube-vip](/2021/04/23/K3s-ha-kube-vip.html).

It should be like that:
```
root@p1:~# ls -l /var/lib/rancher/k3s/server/manifests/
total 24
-rw------- 1 root root 1092 Jun 25 12:36 ccm.yaml
-rw------- 1 root root 4173 Jun 25 12:36 coredns.yaml
-rw------- 1 root root 2740 Jun 25 12:36 local-storage.yaml
drwx------ 2 root root 4096 Jun 12 13:34 metrics-server
-rw------- 1 root root 1039 Jun 25 12:36 rolebindings.yaml
```

To join to k3s cluster ip-address of cluster (in my case it's 192.168.8.10) and secure token has to be confirmed. Token can be known from any of left masters:
```
root@p2:~# awk -F: '{print $4}' /var/lib/rancher/k3s/server/token
mysupersecuretoken
```

Startup script has to be updated to join new master to k3s cluster
```
vi /etc/systemd/system/k3s.service
# ExecStart=/usr/local/bin/k3s server --no-deploy servicelb --no-deploy traefik
ExecStart=/usr/local/bin/k3s server --server https://192.168.8.10:6443 --token mysupersecuretoken --no-deploy servicelb --no-deploy traefik
```

After updating systemctl should be reloaded:
```
systemctl daemon-reload
```

### Join master node to cluster
Just start k3s service and monitor system logs, in few minutes new master should join the cluster:
```
systemctl start k3s

roma@mair ~ % kubectl get node
NAME   STATUS   ROLES         AGE   VERSION
p1     Ready    etcd,master   61s   v1.19.8+k3s1
p2     Ready    etcd,master   12d   v1.19.8+k3s1
p3     Ready    etcd,master   12d   v1.19.8+k3s1
p4     Ready    <none>        12d   v1.19.8+k3s1
```
Wait for 10 minutes and then drain new master as k3s should be stopped to return back startup script there.
```
kubectl drain p1 --delete-local-data --ignore-daemonsets --force
systemctl stop k3s
vi /etc/systemd/system/k3s.service
ExecStart=/usr/local/bin/k3s server --no-deploy servicelb --no-deploy traefik

systemctl daemon-reload
```

After start starting k3s back everything should work as usual:
```
systemctl start k3s
```
Just don't forget to mark it as schedulable back again:
```
kubectl uncordon p1
```
