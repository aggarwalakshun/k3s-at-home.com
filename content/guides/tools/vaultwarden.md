+++
date = '2026-02-27T15:17:47+05:30'
draft = false
title = 'Vaultwarden'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})

## Vaultwarden
Vaultwarden is an open-source password manager that allows you to securely store and manage your passwords.

## Installation
1. Create the following directory structure for Vaultwarden:
  ```
    vaultwarden/
    ├── ingrress.yml
    ├── pvc.yml
    ├── svc.yml
    └── vaultwarden.yml
  ```
2. Add the following content to `ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: vw-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - vw.example.com
      secretName: vw-tls
    rules:
    - host: vw.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: vaultwarden-service
              port:
                number: 80
  ```
3. Add the following content to `pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: vaultwarden-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 1Gi
    storageClassName: longhorn
  ```
4. Add the following content to `vaultwarden.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: vaultwarden
    namespace: tools
  spec:
    strategy:
      type: Recreate
    replicas: 1
    selector:
      matchLabels:
        app: vaultwarden
    template:
      metadata:
        labels:
          app: vaultwarden
      spec:
        containers:
        - name: vaultwarden
          image: vaultwarden/server:1.35.4
          ports:
          - containerPort: 80
          env:
          - name: SIGNUPS_ALLOWED
            value: "false"
          volumeMounts:
          - name: data
            mountPath: /data/
        volumes:
        - name: data
          persistentVolumeClaim:
            claimName: vaultwarden-longhorn
  ```
5. Add the following content to `svc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: vaultwarden-service
    namespace: tools
  spec:
    selector:
      app: vaultwarden
    ports:
    - port: 80
      targetPort: 80
  ```
7. Commit and push the changes to your Git repository. Flux will automatically deploy Vaultwarden to your Kubernetes cluster.
