+++
date = '2026-02-26T08:05:43+05:30'
draft = false
title = 'Sealed Secrets'
weight = 7
+++

# Prerequisites
This guide assumes you have set up FluxCD in your Kubernetes cluster. If you have not already done so, please refer to the [FluxCD]({{% ref "guides/fluxcd.md" %}}) guide for installation instructions.

# Sealed Secrets
Sealed Secrets is a Kubernetes controller and tool that allows you to encrypt your secrets into "sealed secrets" that are safe to store in version control. 

## Installation
1. Create the following directory structure for Sealed Secrets:
   ```
    kube-system/
    └── sealed-secrets/
        ├── helmrelease.yml
        └── helmrepository.yml
    ```
2. Add the following content to `kube-system/sealed-secrets/helmrepository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: sealed-secrets
      namespace: flux-system
    spec:
      interval: 6h
      url: https://bitnami-labs.github.io/sealed-secrets
    ```
3. Add the following content to `kube-system/sealed-secrets/helmrelease.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: sealed-secrets
      namespace: kube-system
    spec:
      chart:
        spec:
          chart: sealed-secrets
          sourceRef:
            kind: HelmRepository
            name: sealed-secrets
            namespace: flux-system
          version: '>=1.15.0-0'
      install:
        crds: Create
      interval: 6h
      releaseName: sealed-secrets-controller
      upgrade:
        crds: CreateReplace
      values:
        networkPolicy:
          enabled: true
    ```

## Usage
To create a sealed secret, you can use the `kubeseal` CLI tool. Insall it on your local machine using the following command:
```bash
export KUBESEAL_VERSION='0.35.0' && \
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz" && \
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal && \
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```
