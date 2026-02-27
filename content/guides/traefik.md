+++
date = '2026-02-26T13:45:53+05:30'
draft = false
title = 'Traefik'
Weight = 3
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})

## Traefik
Traefik is a modern, dynamic reverse proxy and load balancer that integrates seamlessly with Kubernetes. It provides features such as automatic SSL certificate management, HTTP/2 support, and advanced routing capabilities, making it an ideal choice for managing ingress traffic in a Kubernetes cluster.

## Installation
1. Create the following directory structure for Traefik:
    ```
    traefik/
    ├── helmrelease.yml
    └── helmrepository.yml
    ```
2. Add the following content to `traefik/helmrepository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: traefik
      namespace: flux-system
    spec:
      interval: 6h
      url: https://traefik.github.io/charts
    ```
3. Add the following content to `traefik/helmrelease.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: traefik
      namespace: kube-system
    spec:
      chart:
        spec:
          chart: traefik
          sourceRef:
            kind: HelmRepository
            name: traefik
            namespace: flux-system
          version: '39.0.2'
      install:
        crds: Create
      interval: 6h
      releaseName: traefik
      upgrade:
        crds: CreateReplace
      values:
        logs:
          access:
            enabled: true

        deployment:
          enabled: true
          kind: DaemonSet
          dnsPolicy: ClusterFirstWithHostNet
        updateStrategy:
          type: RollingUpdate
          rollingUpdate:
            maxUnavailable: 1
            maxSurge: 0

        hostNetwork: true

        service:
          enabled: false

        securityContext:
          capabilities:
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
          runAsGroup: 0
          runAsNonRoot: false
          runAsUser: 0
          fsGroup: 0

        ports:
          web:
            port: 80
            exposedPort: 80
            protocol: TCP
            expose:
              default: true
            http:
              encodedCharacters:
                allowEncodedSlash: true
                allowEncodedQuestionMark: true

          websecure:
            port: 443
            exposedPort: 443
            protocol: TCP
            expose:
              default: true
            http:
              encodedCharacters:
                allowEncodedSlash: true
                allowEncodedQuestionMark: true

          ssh:
            port: 22
            exposedPort: 22
            protocol: TCP
            expose:
              default: true

          metrics:
            port: 9101
            exposedPort: 9101
            protocol: TCP
            expose:
              default: false

          traefik:
            port: 8081
            exposedPort: 8081
            protocol: TCP
            expose:
              default: false

        providers:
          kubernetesCRD: {}
          kubernetesIngress: {}
      ```
4. Commit and push the changes to your Git repository. FluxCD will automatically deploy Traefik to your Kubernetes cluster based on the HelmRelease configuration you provided.
