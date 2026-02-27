+++
date = '2026-02-27T11:09:49+05:30'
draft = false
title = 'Intel'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'gpu-operator'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Cert-Manager]({{% ref "guides/cert-manager.md" %}})
- Label nodes with `intel.feature.node.kubernetes.io/gpu=true` to indicate that they have Intel GPUs. You can do this using the following command:
    ```bash
    kubectl label nodes <node-name> intel.feature.node.kubernetes.io/gpu=true
    ```

## Intel GPU Operator
Provides support for Intel GPUs in kubernetes clusters.

## Installation
1. Create the following directory structure for Intel GPU Operator:
    ```
    intel-gpu-operator/
    ├── intel-device-operator.yml
    ├── intel-plugin-operator.yml
    └── intel-plugin-repo.yml
    ```
2. Add the following content to `intel-gpu-operator/intel-device-operator.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: device-plugin-operator
      namespace: gpu-operator
    spec:
      interval: 24h
      chart:
        spec:
          chart: intel-device-plugins-operator
          version: "0.35.0"
          sourceRef:
            kind: HelmRepository
            name: intel
            namespace: flux-system
          interval: 24h
      install:
        remediation:
          retries: 3
      upgrade:
        remediation:
          retries: 3
    ```
3. Add the following content to `intel-gpu-operator/intel-plugin-operator.yml
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: gpu-device-plugin
      namespace: gpu-operator
    spec:
      interval: 6h
      chart:
        spec:
          chart: intel-device-plugins-gpu
          version: "0.35.0"
          sourceRef:
            kind: HelmRepository
            name: intel
            namespace: flux-system
          interval: 6h
      install:
        remediation:
          retries: 3
      upgrade:
        remediation:
          retries: 3
      values:
        sharedDevNum: 4
        nodeFeatureRule: false
        nodeSelector:
          intel.feature.node.kubernetes.io/gpu: 'true'
    ```
4. Add the following content to `intel-gpu-operator/intel-plugin-repo.yml
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: intel
      namespace: flux-system
    spec:
      interval: 6h
      url: https://intel.github.io/helm-charts
    ```
5. Commit and push the changes to your Git repository.
