+++
date = '2026-02-27T12:37:25+05:30'
draft = false
title = 'Metallb'
weight = 8
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})

## Metallb
Metallb is a load-balancer implementation for bare metal Kubernetes clusters.

## Installation
1. Create the following directory structure for Metallb:
   ```
    metallb/
    ├── l2-advertisement.yml
    ├── helmrelease.yml
    ├── helmrepository.yml
    └── ip-pool.yml
   ```
2. Add the following content to `metallb/l2-advertisement.yml`:
   ```yaml
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: k3s-lb-pool
      namespace: metallb-system
    spec:
      ipAddressPools:
      - ip-pool
    ```
3. Add the following content to `metallb/helmrepository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: metallb
      namespace: flux-system
    spec:
      interval: 6h
      url: https://metallb.github.io/metallb
    ```
4. Add the following content to `metallb/helmrelease.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: metallb
      namespace: metallb-system
    spec:
      interval: 6h
      chart:
        spec:
          chart: metallb
          version: "0.15.3"
          sourceRef:
            kind: HelmRepository
            name: metallb
            namespace: flux-system
          interval: 6h
      install:
        createNamespace: true
      upgrade:
        remediation:
          remediateLastFailure: true
    ```
5. Add the following content to `metallb/ip-pool.yml`:
    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: ip-pool
      namespace: metallb-system
    spec:
      addresses:
      - 192.168.1.201-192.168.1.250
    ```
6. Commit and push the changes to your Git repository.
