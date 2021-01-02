---
layout: post
title:  "Deploying K3s cluster on Virtualbox"
date:   2021-01-02 20:00:00 +0200
categories: kubernetes
tags: kubernetes virtualbox k3s vagrant
---

# Overview
This post contains instruction about how to deploy three nodes K3s cluster on Virtualbox for testing purposes.

# Story
HA (high available) systems in modern IT world is *de facto* standard and I manage them for years at work. 
However I always wanted to have my own HA cluster at home to have well structured and robust platform for my own purposes + at the same time it would be good to develop my expertise with Kubernetes, containers and IaC (infrastructure as a code). 

So I've started to explore what way I can build kubernetes at home and what I need for that. Don't remember how exactly, but I've discovered [Jeff Geerling's site](https://www.jeffgeerling.com), his "Raspberry Pi Cluster" series and my choice was made - k3s on raspberry pi. 

I've bought three raspberry pi 4's with 4GB of RAM and built my kubernetes cluster as per Jeff's instruction. 
It worked fine but it was not good when I needed to change something on a cluster - I needed to test it first somewhere before to push to *production* and at the same time kubernetes is not the most simple thing when you just start - you break & build it from scratch many times.

So this is how I've come to fact that I need test kubernetes cluster which would be similar to my *production* one (k3s on raspberries) and this post is about how to use Virtualbox as a platform for k3s cluster.

# Instruction
## Prepare 4x VMs
With a help of [Vagrant](https://en.wikipedia.org/wiki/Vagrant_(software)) and [Virtualbox](https://en.wikipedia.org/wiki/VirtualBox) the following Vagrantfile should be created to build 4x VMs based on Debian 10
```
$ cat Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian10"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = "2"
    vb.linked_clone = true
    vb.customize ['modifyvm', :id, '--audio', 'none']
  end
  
  boxes = [
    { :name => "kube0", :ip => "192.168.8.20" },
    { :name => "kube1", :ip => "192.168.8.21" },
    { :name => "kube2", :ip => "192.168.8.22" },
    { :name => "kube3", :ip => "192.168.8.23" },
  ]
  
  boxes.each_with_index do |opts, index|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name] + ".cluster.test"
      config.vm.network :private_network, ip: opts[:ip]
    end
  end
  
end
```
* kube0 - just VM to rebuild cluster with a help of Ansible
* kube1 - master node
* kube2 & kube3 - worker nodes

## Build K3s cluster with Ansible
Install Ansible and git on kube0
```
$ vagrant ssh kube0
$ sudo apt-get install git ansible -y
```
Clone [k3s via Ansible](https://github.com/rancher/k3s-ansible) repository and update configuration:
```
$ git clone https://github.com/rancher/k3s-ansible
$ mkdir k3s-ansible/inventory/group_vars
$ vi k3s-ansible/inventory/group_vars/all.yml
---
k3s_version: v1.17.5+k3s1
ansible_user: vagrant
systemd_dir: /etc/systemd/system
{% raw  %}master_ip: "{{ hostvars[groups['master'][0]]['ansible_host'] | default(groups['master'][0]) }}"
extra_server_args: "--node-ip={{ hostvars[groups['master'][0]]['ansible_host'] | default(groups['master'][0]) }} --flannel-iface=eth1"{% endraw %}
extra_agent_args: "--flannel-iface=eth1"
```
* ansible_user should be vagrant as it's default account which is used when you build VM with Vagrant
* extra_server_args and extra_agent_args variables should be updated as specified otherwise eth0 (10.0.2.15) will be used on each new kube-VM and it won't work

```
$ vi k3s-ansible/inventory/hosts.ini 
[master]
192.168.8.21

[node]
192.168.8.22
192.168.8.23

[k3s_cluster:children]
master
node
```
* kube1 - master node
* kube2 & kube3 - worker nodes

Run K3s deployment:
```
$ cd k3s-ansible
$ ansible-playbook site.yml -i inventory/hosts.ini
```
After all steps of ansible playbook are completed you should have three nodes kubernetes cluster ready for testing. 
Yes, it won't be ARM architecture but at least it will be very similar to everything else - three nodes with the same version of k3s and its components.

## Check deployed k3s cluster
Login to kube1 (master) and check deployed kubernetes components:
```
$ vagrant ssh kube1
vagrant@kube1:~$ sudo su -
root@kube1:~# cp /home/vagrant/.kube/config /root/.kube/
root@kube1:~# kubectl get pod -A -o wide
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
kube-system   helm-install-traefik-fkf2f                0/1     Completed   0          6h21m   10.42.2.4    kube1   <none>           <none>
kube-system   local-path-provisioner-58fb86bdfd-8ch5s   1/1     Running     1          6h21m   10.42.2.9    kube1   <none>           <none>
kube-system   metrics-server-6d684c7b5-279lm            1/1     Running     1          6h21m   10.42.2.8    kube1   <none>           <none>
kube-system   svclb-traefik-c4bcb                       2/2     Running     2          6h21m   10.42.2.11   kube1   <none>           <none>
kube-system   coredns-6c6bb68b64-fjkjc                  1/1     Running     1          6h21m   10.42.2.10   kube1   <none>           <none>
kube-system   svclb-traefik-tlpj7                       2/2     Running     2          6h21m   10.42.1.9    kube2   <none>           <none>
kube-system   svclb-traefik-2mzb2                       2/2     Running     2          6h21m   10.42.0.5    kube3   <none>           <none>
kube-system   traefik-7b8b884c8-zgddw                   1/1     Running     1          6h21m   10.42.0.4    kube3   <none>           <none>
root@kube1:~# kubectl get node -o wide
NAME    STATUS   ROLES    AGE     VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
kube1   Ready    master   6h21m   v1.17.5+k3s1   192.168.8.21   <none>        Debian GNU/Linux 10 (buster)   4.19.0-6-amd64   containerd://1.3.3-k3s2
kube2   Ready    <none>   6h21m   v1.17.5+k3s1   192.168.8.22   <none>        Debian GNU/Linux 10 (buster)   4.19.0-6-amd64   containerd://1.3.3-k3s2
kube3   Ready    <none>   6h21m   v1.17.5+k3s1   192.168.8.23   <none>        Debian GNU/Linux 10 (buster)   4.19.0-6-amd64   containerd://1.3.3-k3s2
```

# Useful
If you want to destroy your test k3s cluster just run the following from kube0
```
$ ansible-playbook reset.yml -i inventory/hosts.ini
```

# Links
* [Raspberry Pi Cluster Episode 3 - Installing K3s Kubernetes on the Turing Pi](https://www.jeffgeerling.com/blog/2020/installing-k3s-kubernetes-on-turing-pi-raspberry-pi-cluster-episode-3)
* [Raspberry Pi Cluster Episode 4 - Minecraft, Pi-hole, Grafana and More!](https://www.jeffgeerling.com/blog/2020/raspberry-pi-cluster-episode-4-minecraft-pi-hole-grafana-and-more)