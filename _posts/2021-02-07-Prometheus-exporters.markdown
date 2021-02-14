---
layout: post
title:  "Prometheus exporters"
date:   2022-02-07 10:00:00 +0200
tags: prometheus exporter snmp arm
---
# Overview
It is possible to add to Prometheus modules (exporters) which would collect metrics from external objects via non-prometheus protocols (snmp as example) and then to provide them to Prometheus as usual targets.

Today I'm using two exporters arm-exporter (to get cpu temperature or raspberry pi cpu) and snmp-exporter (to get network statistics from my network devices).

This is how I installed & configure them in my k3s cluster.

# snmp-exporter


# arm-exporter
Built from [Carlos Eduardo's Cluster Monitoring Repo](https://github.com/carlosedp/cluster-monitoring) with make and then update armexporter-serviceMonitor.yaml updated as per the following (to add to labels `release: prometheus`):
```
cat armexporter-serviceMonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: arm-exporter
    release: prometheus
  name: arm-exporter
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 30s
    port: https
    relabelings:
    - action: replace
      regex: (.*)
      replacement: $1
      sourceLabels:
      - __meta_kubernetes_pod_node_name
      targetLabel: instance
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
  jobLabel: arm-exporter-exporter
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: arm-exporter
```
Then just apply all armexporter-files:
```
ls -la
total 40
drwxr-xr-x 2 root root  4096 Jan 17 23:09 .
drwxrwsrwx 3 1030 users 4096 Jan 31 10:32 ..
-rw-r--r-- 1 root root   263 Jan 17 21:51 armexporter-clusterRoleBinding.yaml
-rw-r--r-- 1 root root   282 Jan 17 21:51 armexporter-clusterRole.yaml
-rw-r--r-- 1 root root  1736 Jan 17 21:51 armexporter-daemonset.yaml
-rw-r--r-- 1 root root    91 Jan 17 21:51 armexporter-serviceAccount.yaml
-rw-r--r-- 1 root root   671 Jan 17 23:09 armexporter-serviceMonitor.yaml
-rw-r--r-- 1 root root   244 Jan 17 21:51 armexporter-service.yaml

kubectl apply -f arm*

kubectl get pod -n monitoring | grep arm
arm-exporter-48rps                                        2/2     Running   0          27d
arm-exporter-mgvq4                                        2/2     Running   2          27d
arm-exporter-rmmc8                                        2/2     Running   2          27d
```
After few minutes prometheus will see new exporter and they should be available as targets - one target per each Kubernetes node:
![Arm-exporter targets in Prometheus](/blog/assets/arm-exporter-prometheus.png)
Next step is to create graph in Grafana:
![Arm-exporter data in Grafana](/blog/assets/arm-exporter-grafana.png)


