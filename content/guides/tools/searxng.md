+++
date = '2026-02-27T15:14:59+05:30'
draft = false
title = 'Searxng'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Searxng
Searxng is a privacy-respecting metasearch engine that aggregates results from various search engines.

## Installation
1. Create the following directory structure for Searxng:
  ```
    searxng/
    ├── ingress.yml
    ├── svc.yml
    ├── pvc.yml
    └── searxng.yml
  ```
2. Add the following content to `searxng.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: searxng
    namespace: tools
  spec:
    strategy:
      type: Recreate
    replicas: 1
    selector:
      matchLabels:
        app: searxng
    template:
      metadata:
        labels:
          app: searxng
      spec:
        containers:
        - name: searxng
          image: searxng/searxng@sha256:edf110a2816d8963949d03879c72a7e19c221b5f7bfb7952a33ae073f96ccb18
          ports:
          - containerPort: 8080
          env:
          - name: "INSTANCE_NAME"
            value: "searxng"
          - name: BASE_URL
            value: "searxng.example.com"
          volumeMounts:
          - name: searxng
            mountPath: /etc/searxng
        volumes:
        - name: searxng
          persistentVolumeClaim:
            claimName: searxng-longhorn
  ```
3. Add the following content to `svc.yml`:
  ```yaml  
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: searxng-service
    namespace: tools
  spec:
    selector:
      app: searxng
    ports:
    - port: 8080
      targetPort: 8080
  ```
4. Add the following content to `ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: searxng-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - sear.example.com
      secretName: homepage-tls
    rules:
    - host: sear.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: searxng-service
              port:
                number: 8080
  ```
5. Add the following content to `pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: searxng-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 100Mi
    storageClassName: longhorn
  ```
6. Commit and push the files to the repository. Flux will automatically deploy Searxng to the cluster.
