+++
date = '2026-02-27T14:37:37+05:30'
draft = false
title = 'Uptime Kuma'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authello]({{% ref "guides/tools/authelia.md" %}})

## Uptime Kuma
Uptime Kuma is a self-hosted monitoring tool that allows you to keep track of the uptime and performance of your services.

## Installation
1. Create the following directory structure for Uptime Kuma:
    ```
    uptime-kuma/
    ├── ingress.yml
    ├── pvc.yml
    ├── svc.yml
    └── uptime-kuma.yml
    ```
2. Add the following content to the `ingress.yml` file:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: uptime-kuma-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - status.example.com
        secretName: uptime-kuma-tls
      rules:
      - host: status.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: uptime-kuma-service
                port:
                  number: 3001
    ```
3. Add the following content to the `pvc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: uptime-kuma-longhorn
      namespace: monitoring
    spec:
      resources:
        requests:
          storage: 1Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
    ```
4. Add the following content to the `svc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: uptime-kuma-service
      namespace: monitoring
    spec:
      selector:
        app: uptime-kuma
      type: ClusterIP
      ports:
      - port: 3001
        targetPort: 3001
        name: http
    ```
5. Add the following content to the `uptime-kuma.yml` file:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: uptime-kuma
      namespace: monitoring
    spec:
      replicas: 1
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app: uptime-kuma
      template:
        metadata:
          labels:
            app: uptime-kuma
        spec:
          containers:
          - name: uptime-kuma
            image: louislam/uptime-kuma:2.1.3
            ports:
            - containerPort: 3001
              name: http
            volumeMounts:
            - name: uptime-kuma-data
              mountPath: /app/data
          volumes:
          - name: uptime-kuma-data
            persistentVolumeClaim:
              claimName: uptime-kuma-longhorn
    ```
6. Commit and push the changes to your Git repository. Flux will deploy Uptime Kuma to your cluster.
