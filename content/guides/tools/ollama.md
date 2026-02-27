+++
date = '2026-02-27T15:00:19+05:30'
draft = false
title = 'Ollama'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Nvidia GPU Operator]({{% ref "guides/gpu-operator/nvidia.md" %}})

## Ollama
Ollama is a platform for running and managing large language models (LLMs) on your own infrastructure.

## Installation
1. Create the following directory structure for Ollama:
  ```
    ollama/
    ├── pvc.yml
    ├── helmrepository.yml
    └── helmrelease.yml
  ```
2. Add the following content to `pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: ollama-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 10Gi
    storageClassName: longhorn
  ```
3. Add the following content to `helmrepository.yml`:
  ```yaml
  ---
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: ollama
    namespace: tools
  spec:
    interval: 6h
    chart:
      spec:
        chart: ollama
        version: "1.45.0"
        sourceRef:
          kind: HelmRepository
          name: ollama
          namespace: flux-system
        interval: 6h
    install:
      remediation:
        retries: 3
    upgrade:
      remediation:
        retries: 3
    values:
      ollama:
        gpu:
          enabled: true
          type: nvidia
      service:
        type: ClusterIP
      runtimeClassName: nvidia
      persistentVolume:
        enabled: true
        existingClaim: ollama-longhorn
  ```
4. Add the following content to `helmrepository.yml`:
  ```yaml
  ---
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: ollama
    namespace: flux-system
  spec:
    interval: 6h
    url: https://otwld.github.io/ollama-helm/
  ```
5. Commit and push the files to your Git repository. Flux will automatically deploy the resources to your cluster.
