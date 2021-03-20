---
layout: post
title: "NFS Subdirectory External Provisioner as Storageclass in K3s"
date: 2021-02-20 10:00:00 +0200
tags: k3s storage nfs
published: true
---
# Overview
K3s is small because a lot different not-mandatory components of Kubernetes are not included by default. As example K3s has only one available storageclass - local-path.
```
# kubectl get storageclass 
NAME                   PROVISIONER                                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path                          Delete          WaitForFirstConsumer   false                  33d
```

Default local-path is more than enough to play around with storagesclass, persistent volumes and persistent volume claims.

When there is cluster of several nodes, volumes (which are used by pods) have to be external to nodes, otherwise each pod will have different data when it runs on different node (as it will use local volume). In home environment the most simple external storage can be NAS (network attached storage) accessible via NFS/CIFS/AFS protocols. So the same volume which is accessible via NFS can be mounted on all nodes and in that case local-path storageclass can be used still. 

I started from there using NAS via NFS and it worked fine for most of the cases. However it means you need to make node configuration external to K3s - /etc/fstab has to be updated and managed, required mount modules have to be install on all nodes and the same volume has to be mounted on all nodes and if something goes wrong - you need to investigate the problem outside of K3s.

So next step was - discovery of *nfs-client*, it brings nfs-client as new storageclass into kubernetes cluster and allows to use it for persistent volume claims (pvc). nfs-client class can create (and manage afterwards) NFS volumes based on defined configuration and specified pvc-requests. 

# Installation
Before to install nfs-client into kubernetes cluster your NFS server has to be configured the following way:
- NFS should be accessible to all nodes via ip or hostname (192.168.8.20 on example below)
- each node has to have write permissions to nfs-path which will be used as base for all kubernetes nfs-volumes (/mnt/kubcluster on example below)
 
The easiest way to install nfs-client is to use helm and create storage namespace for it before:
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm search repo | grep nfs-subdir
nfs-subdir-external-provisioner/nfs-subdir-exte...	4.0.4        	4.0.0                  	nfs-subdir-external-provisioner is an automatic...

kubectl create namespace storage

helm install -n storage --set nfs.server=192.168.8.20 --set nfs.path=/mnt/kubcluster storage nfs-subdir-external-provisioner/nfs-subdir-external-provisioner
```
As result you should see new storageclass available and pod running:
```
# kubectl get pod -n storage
NAME                                                     READY   STATUS    RESTARTS   AGE
storage-nfs-subdir-external-provisioner-dd4dbdf5-5rmrw   1/1     Running   0          9m57s
# kubectl get storageclass
NAME                   PROVISIONER                                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path                                   Delete          WaitForFirstConsumer   false                  6d17h
nfs-client             cluster.local/storage-nfs-subdir-external-provisioner   Delete          Immediate              true                   10m
```

# Usage
Usage is very simple - just specify `volume.beta.kubernetes.io/storage-class: nfs-client` as annotations in metadata for pvc you create.
As example the following yaml-file will create grafana-nfs-pvc which will automatically create pv with automatically generated name:
```
# cat monitoring/grafana-nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "monitoring"
  name: "grafana-nfs-pvc"
  annotations:
    volume.beta.kubernetes.io/storage-class: nfs-client
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "50Mi"

# kubectl get pvc grafana-nfs-pvc -n monitoring
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
grafana-nfs-pvc   Bound    pvc-1f903c25-823b-466d-b226-800c1e0a05d2   50Mi       RWO            nfs-client     27d

# kubectl get pv pvc-1f903c25-823b-466d-b226-800c1e0a05d2
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pvc-1f903c25-823b-466d-b226-800c1e0a05d2   50Mi       RWO            Delete           Bound    monitoring/grafana-nfs-pvc   nfs-client              27d
```

This is what exists on NFS-server:
```
# ls -l /mnt/kubcluster | grep pvc-1f903c25-823b-466d-b226-800c1e0a05d2
drwxrwxrwx  4 root root 4096 Feb 20 12:38 monitoring-grafana-nfs-pvc-pvc-1f903c25-823b-466d-b226-800c1e0a05d
```

If pvc is deleted volume will be renamed into archived, but data will be left on NFS share. It is managed by Reclaim policy setting in storageclass ([for more details](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)).
