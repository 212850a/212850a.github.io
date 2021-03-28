---
layout: post
title: "How to import Grafana dashboards via k8s-sidecar"
date: 2021-03-28 10:00:00 +0200
tags: k3s k8s grafana dashboard
published: true
---
## Overview
After Grafana is installed as per [kube-prometheus-stack](/2021/01/20/Installing-Prometheus-with-Helm.html) there is a way to import saved before dashboards via [k8s-sidecar](https://github.com/kiwigrid/k8s-sidecar) container which is running on the same pod as Grafana itself:
```
kubectl get -n monitoring pod | grep grafana
prometheus-grafana-57f78bfdb-9xq96                        2/2     Running   0          32m

kubectl describe pod prometheus-grafana-57f78bfdb-9xq96 -n monitoring | egrep -A 25 "^Containers:" | grep "Image:"
    Image:          quay.io/kiwigrid/k8s-sidecar:1.10.7
    Image:          grafana/grafana:7.4.5

kubectl describe pod prometheus-grafana-57f78bfdb-9xq96 -n monitoring | grep -A 17 "grafana-sc-dashboard:"
  grafana-sc-dashboard:
    Container ID:   containerd://088f282a72b2bf512a2dcaf5040658fca6b557d2406b0d8c7b1f5dc52284ac0d
    Image:          quay.io/kiwigrid/k8s-sidecar:1.10.7
    Image ID:       quay.io/kiwigrid/k8s-sidecar@sha256:18feb3906286814364b2f42eeaf82649b4847a4d0a779222a613c79c9da7ad87
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 28 Mar 2021 10:38:25 +0300
    Ready:          True
    Restart Count:  0
    Environment:
      METHOD:    
      LABEL:     grafana_dashboard
      FOLDER:    /tmp/dashboards
      RESOURCE:  both
    Mounts:
      /tmp/dashboards from sc-dashboard-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from prometheus-grafana-token-pkk2s (ro)

```
High level plan is:
- Export desired version of Grafana dashboard and save it as json-file
- Prepare ConfigMap based on exported before json-formatted dashboard
- Apply ConfigMap to existing monitoring namespace k8s-sidecar container to import it to Grafana

## Export desired grafana dashboard(s)
Firstly prepare version of dashboard which you want to save for future, select `Share -> Export -> Save to file` and save it as grafana-dashboard-k3s.json (as example):

![grafana-export-dashboard-as-file](/assets/grafana-export-dashboard-as-file.png)

## Prepare ConfigMap yaml-file
Based on exported dashboard as json-file ConfigMap yaml-file should be prepared. It should look like:
```
cat > grafana-dashboard-k3s.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-k3s
  labels:
    grafana_dashboard: "1"
  namespace: monitoring
data:
  grafana-dashboard-k3s.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "description": "Monitor a Kubernetes cluster using Prometheus TSDB.  Shows overall cluster CPU / Memory / Disk usage as well as individual pod statistics. ",
      "editable": true,
      "gnetId": 162,
      "graphTooltip": 1,
      "id": 25,
        ...
      "timezone": "browser",
      "title": "Kubernetes cluster monitoring (via Prometheus)",
      "uid": "hgN5WikRz",
      "version": 11
}
    }

```
As example - [grafana-dashboard-k3s.yaml](https://github.com/212850a/home-kube/blob/main/roles/monitoring/files/grafana-dashboard-k3s.yaml), it's based on [kubernetes-cluster-dashboard.json](https://github.com/carlosedp/cluster-monitoring/blob/master/grafana-dashboards/kubernetes-cluster-dashboard.json) from [Cluster Monitoring stack for ARM / X86-64 platforms](https://github.com/carlosedp/cluster-monitoring).

## Apply ConfigMap to monitoring namespace
After applying prepared configmap to monitoring namespace you should see the following logs for grafana-sc-dashboard container:

```
kubectl -f grafana-dashboard-router.yaml apply

kubectl -n monitoring logs prometheus-grafana-57f78bfdb-9xq96 grafana-sc-dashboard
[2021-03-28 07:46:08] Working on configmap monitoring/grafana-dashboard-k3s
[2021-03-28 07:46:08] File in configmap grafana-dashboard-k3s.json ADDED

```
Straight after that added dashboard should be visible in Grafana under `Dashboards -> Manage`.

## Links
* [Automate GRAFANA Dashboard Import Process](https://codevalue.com/grafana/)
* [Sidecar for dashboards](https://github.com/grafana/helm-charts/tree/main/charts/grafana#sidecar-for-dashboards)
