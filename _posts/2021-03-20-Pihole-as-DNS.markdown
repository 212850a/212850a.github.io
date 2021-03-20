---
layout: post
title: "Pihole as DNS service"
date: 2021-03-20 10:00:00 +0200
tags: pihole dns metallb k3s k8s
published: true
---
## Overview
DNS as service is one of the core ones, so it's really important and useful in most cases to have control about how it works for you. Pihole is really good implementation of DNS service with ability to block requests to advertisement, get powerful statistics and control about how it works for you. For instance you can configure it (via DHCP settings) on all local devices and additionally to forward all not blocked DNS requests to OpenDNS service for additional filtering. Whitelists and blocklists are available there either.
![pihole-dashboard](/assets/pihole-dashboard.png)

## Deployment in k3s
- Reserve ip-address on MetalLB loadbalancer for Pihole
- Create namespace for Pihole
- Create persistent volume claim for future Pihole settings and database
- Deploy Pihole using helm

### Reserve ip-address on MetalLB loadbalancer for Pihole
I recommend to reserve some specific ip-address for DNS Pihole service on MetalLB configuration as it should be populated then via DHCP onto devices from your local network. To do that existing MetalLB configuration file should be updated with pihole-services section. In example below 192.168.8.11 is reserved for Pihole Loadbalancer service on [MetalLB](/2021/01/15/Metallb-as-LoadBalancer-for-K3s.html):
```
cat metallb/config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.8.2-192.168.8.10
    - name: pihole-services
      protocol: layer2
      addresses:
      - 192.168.8.11-192.168.8.11

kubectl -f metallb/config.yaml apply
```

### Create namespace for Pihole
It's better to have separate namespace for Pihole as it will make improve management of it.
```
kubectl create namespace pihole
```

### Create persistent volume claim for Pihole
[NFS-client storageclass](/2021/02/20/NFS-Client.html) should be defined before the following configuration to be applied:

```
cat pihole-nfs-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "pihole"
  name: "pihole-nfs-pvc"
  annotations:
    volume.beta.kubernetes.io/storage-class: nfs-client
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "500Mi"


kubectl -f pihole-nfs-pvc.yaml
```

### Deploy Pihole using helm
Add mojo2600 helm repository first
```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
```
Prepare values-file for Pihole helm chart. Based on already created above objects the following values-file should be working:
```
cat pihole-values.yaml
---
persistentVolumeClaim:
  enabled: true
  existingClaim: "pihole-nfs-pvc"
serviceWeb:
  type: LoadBalancer
  annotations:
    metallb.universe.tf/address-pool: pihole-services
    metallb.universe.tf/allow-shared-ip: pihole-svc
serviceDns:
  type: LoadBalancer
  annotations:
    metallb.universe.tf/address-pool: pihole-services
    metallb.universe.tf/allow-shared-ip: pihole-svc
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
# If using in the real world, set up admin.existingSecret instead.
adminPassword: admin
``` 
Install helm chart using defined values
```
helm install --namespace pihole --values pihole-values.yaml pihole mojo2600/pihole
```

## Links
- [Raspberry Pi Cluster Episode 4 - Minecraft, Pi-hole, Grafana and More!](https://www.jeffgeerling.com/blog/2020/raspberry-pi-cluster-episode-4-minecraft-pi-hole-grafana-and-more)