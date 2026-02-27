+++
date = '2026-02-26T08:20:47+05:30'
draft = false
title = 'Longhorn'
weight = 6
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})
- [Cert-Manager]({{% ref "guides/cert-manager.md" %}})
- Additional packages may be required depending on your Kubernetes environment. Please refer to the [Longhorn documentation](https://longhorn.io/docs/) for specific prerequisites related to your setup.
- Ensure the `dm_crypt` kernel module is loaded:
  ```bash
  sudo modprobe dm-crypt
  ```
- You also need to install these packages:
  ```bash
  sudo apt install cryptsetup dmsetup nfs-common open-iscsi
  ```
- Ingress will be protected with Authelia, so you need to have Authelia installed and configured in your cluster. Please refer to the [Authelia guide]({{% ref "guides/tools/authelia.md" %}}) for installation and configuration instructions.

## Longhorn
Longhorn is a cloud-native distributed block storage solution for Kubernetes. It provides highly available and persistent storage for your applications running in Kubernetes clusters.

## Installation
1. Create the following directory structure for Longhorn:
   ```
    longhorn/
    ├── longhorn/
    │   ├── helmrelease.yml
    │   ├── helmrepository.yml
    │   └── ingress.yml
    └── namespace.yml
   ```

2. Add the following content to `longhorn/namespace.yml`:
   ```yaml
    ---
    kind: Namespace
    apiVersion: v1
    metadata:
      name: longhorn-system
      labels:
        name: longhorn-system
    ```

3. Add the following content to `longhorn/helmrepository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: longhorn
      namespace: flux-system
    spec:
      interval: 6h
      url: https://charts.longhorn.io
    ```
4. Add the following content to `longhorn/helmrelease.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: longhorn
      namespace: longhorn-system
    spec:
      interval: 6h
      chart:
        spec:
          chart: longhorn
          version: "1.10.1"
          sourceRef:
            kind: HelmRepository
            name: longhorn
            namespace: flux-system
          interval: 6h
      install:
        createNamespace: true
      upgrade:
        remediation:
          remediateLastFailure: true
      values:
        persistence:
          defaultClass: false
          reclaimPolicy: Retain
        ingress:
          enabled: false
        service:
          ui:
            type: ClusterIP
      ```
  
5. Add the following content to `longhorn/ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-ingress
      namespace: longhorn-system
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - longhorn.example.com
        secretName: longhorn-tls
      rules:
      - host: longhorn.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
    ```
6. Commit and push the changes to your Git repository. FluxCD will automatically detect the changes and deploy Longhorn to your Kubernetes cluster.
