---
layout: post
title:  "Installing Prometheus stack with Helm"
date:   2021-01-03 20:00:00 +0200
published: true
categories: kubernetes monitoring
tags: kubernetes prometheus k3s grafana
---

## Overview
Not once practice demonstrated to me that if there are some historical statistics about working system it will be much more easier to understand how system works and solve any problem which may happen. With Kubernetes I believe the most first step (before running any application pod) should be installing of monitoring components which would tell you about your cluster as much as possible - to know how busy nodes are in cpu, memory, disk and network IO and so on. 

The most frequently used monitoring system for Kubernetes (and not only for it) these days is Prometheus which is built as cloud native application from the beginning. It can query metrics on remote targets, store collected metric in time series database and even visualize them in web GUI. However very often Prometheus is used in linked with Grafana which is another monitoring tool but it has much more powerful UI for visualizing graphs from collected metrics. When Prometheus is used together with Grafana, Prometheus queries metrics and store them in its database, Grafana is used as UI for graphs only.

![Prometheus architecture](/blog/assets/prometheus_architecture.png)