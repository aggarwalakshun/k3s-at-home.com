+++
date = '2026-02-27T14:18:46+05:30'
draft = false
title = 'Prometheus'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Flux]({{% ref "guides/fluxcd.md" %}})

## Prometheus
Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability.

## Installation
1. Create the following directory structure for Prometheus:
    ```
    prometheus/
    ├── helmrelease.yml
    ├── helmrepository.yml
    └── pvc.yml
    ```
2. Add the following content to the `helmrepository.yml` file:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: prometheus-community
      namespace: flux-system
    spec:
      interval: 6h
      url: https://prometheus-community.github.io/helm-charts
    ```
3. Add the following content to the `helmrelease.yml` file:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: prometheus
      namespace: monitoring
    spec:
      interval: 6h
      chart:
        spec:
          chart: prometheus
          version: "28.9.1"
          sourceRef:
            kind: HelmRepository
            name: prometheus-community
            namespace: flux-system
          interval: 6h
      values:
        server:
          persistentVolume:
            enabled: true
            existingClaim: prometheus-longhorn
          retention: "2d"
          retentionSize: "4GB"
    ```
4. Add the following content to the `pvc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: prometheus-longhorn
      namespace: monitoring
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: longhorn
    ```
5. Commit and push the changes to your git repository. Flux will deploy Prometheus to your cluster.
