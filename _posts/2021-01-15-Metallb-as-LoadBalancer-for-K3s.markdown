---
layout: post
title:  "Using Metallb as LoadBalancer for K3s"
date:   2021-01-15 20:00:00 +0200
published: true
tags: kubernetes matallb k3s loadbalancer
---

## Overview
If you don't have LoadBalancer or Ingress components available you won't be able to access cluster services outside of kubernetes in easy way. 
My K3s cluster didn't have after [installation](/blog/2021/01/02/K3s-on-Virtualbox.html) any of them so I've found Metallb as solution for LoadBalancer.

## Installation
Based on [official guide](https://metallb.universe.tf/installation/) installation is pretty simple.

### Create namespace
Create metallb-system namespace by the following command (alternatively you can download it before to confirm what is inside)
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
```

### Install required components
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
The components in the manifest are:
* the metallb-system/controller deployment. This is the cluster-wide controller that handles IP address assignments.
* the metallb-system/speaker daemonset. This is the component that speaks the protocol(s) of your choice to make the services reachable.
* service accounts for the controller and speaker, along with the RBAC permissions that the components need to function.

This is what it looks like after installation in kubernetes cluster:
```
# kubectl get all -n metallb-system -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
pod/speaker-nlwdt                 1/1     Running   2          6d7h   192.168.8.21   kube1   <none>           <none>
pod/controller-6b8d7594db-s7jcl   1/1     Running   2          6d7h   10.42.0.105    kube1   <none>           <none>
pod/speaker-vbp55                 1/1     Running   2          6d7h   192.168.8.22   kube2   <none>           <none>
pod/speaker-sk5s9                 1/1     Running   2          6d7h   192.168.8.23   kube3   <none>           <none>

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE    CONTAINERS   IMAGES                   SELECTOR
daemonset.apps/speaker   3         3         3       3            3           kubernetes.io/os=linux   6d7h   speaker      metallb/speaker:v0.9.5   app=metallb,component=speaker

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                      SELECTOR
deployment.apps/controller   1/1     1            1           6d7h   controller   metallb/controller:v0.9.5   app=metallb,component=controller

NAME                                    DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES                      SELECTOR
replicaset.apps/controller-6b8d7594db   1         1         1       6d7h   controller   metallb/controller:v0.9.5   app=metallb,component=controller,pod-template-hash=6b8d7594d
```

### Add configuration
The following configuration will allow services to take ip-addresses from 192.168.8.2-192.168.8.10 range:
```
# cat > config.yaml
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

# kubectl apply -f config.yaml
```