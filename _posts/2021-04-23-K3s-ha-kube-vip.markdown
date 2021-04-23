---
layout: post
title: "K3s in HA mode with kube-vip"
date: 2021-04-23 10:00:00 +0200
tags: k3s ha kube-vip
published: true
---
## Overview
After several of months of successful k3s testing in single master mode I’ve started to plan switching to HA mode (multi master mode).

As per [Rancher Docs](https://rancher.com/docs/k3s/latest/en/installation/ha-embedded/) minimal deployment requires at least three masters and k3s should be used older than 1.19.5 as starting from that version etcd is fully enabled (not as experiential feature).

In addition to having [control plane](https://kubernetes.io/docs/concepts/overview/components/) in HA mode I wanted cluster to be available via virtual ip-address (as load balancer is not an option for me). In that case no need to remember about each master ip-address and there is no single point of failure for control plane. 
[Kube-vip](https://kube-vip.io) was selected as component to bring up virtual ip-address on masters.

As with single master mode I used [k3s/k3s-ansible](https://github.com/k3s-io/k3s-ansible/tree/k3s-ha) repository, k3s-ha branch as base for start. But due I planned to used it with conjunction of kube-vip I've forked repository and published all my work as separate branch [k3s-ha-kube-vip](https://github.com/212850a/k3s-ansible/tree/k3s-ha-kube-vip) to my local repository.

## Prerequisites
### Create inventory file
Based on example from inventory/sample create file with your servers (inventory/hosts.ini as example):
```
[master]
192.16.35.11
192.16.35.12
192.16.35.13

[node]
192.16.35.[20:21]

[k3s_cluster:children]
master
node
``` 
### Create variable file
Based on example from inventory/sample inventory/group_vars/all.yml should be created. Here is example for vagrant-configuration:
```
---
k3s_version: v1.19.8+k3s1
ansible_user: vagrant
systemd_dir: /etc/systemd/system
flannel_iface: "eth1"
apiserver_endpoint: "192.16.35.100"
k3s_token: "mysupersecuretoken"
extra_server_args: "--node-ip={{ ansible_eth1.ipv4.address }} --flannel-iface={{ flannel_iface }} --no-deploy servicelb --no-deploy traefik"
extra_agent_args: "--flannel-iface={{ flannel_iface }}"
```
apiserver_endpoint is virtual ip-addresses, which will be created with a help of kube-vip component on each master node.

## Usage
Start provisioning of the cluster using the following command:
```
ansible-playbook site.yml -i inventory/hosts.ini
```
After cluster is built and ansible playbook is completed to get access there just copy /etc/rancher/k3s/k3s.yaml to your ~/.kube/config.
```
% kubectl get nodes
NAME    STATUS   ROLES         AGE     VERSION
kube1   Ready    etcd,master   4h21m   v1.19.8+k3s1
kube2   Ready    etcd,master   4h20m   v1.19.8+k3s1
kube3   Ready    etcd,master   4h20m   v1.19.8+k3s1
kube4   Ready    <none>        4h18m   v1.19.8+k3s1
kube5   Ready    <none>        4h18m   v1.19.8+k3s1
```

To destroy cluster and remove k3s software from servers:
```
ansible-playbook reset.yml -i inventory/hosts.ini
```



