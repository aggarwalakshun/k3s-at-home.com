+++
date = '2026-02-26T18:10:07+05:30'
draft = false
title = 'Seerr'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Seerr
Seerr is an open-source media request management tool that allows users to request movies and TV shows for their media library.

## Installation
1. Create the following directory structure for Seerr:
   ```
    seerr/
    ├── seerr.yml
    ├── seerr-ingress.yml
    ├── seerr-pvc.yml
    └── seerr-svc.yml
    ```

2. Add the following content to `seerr/seerr.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: seerr
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: seerr
      template:
        metadata:
          labels:
            app: seerr
        spec:
          securityContext:
            fsGroup: 1000
            fsGroupChangePolicy: OnRootMismatch
          containers:
          - name: seerr
            image: ghcr.io/seerr-team/seerr:v3.0.1
            securityContext:
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                  - ALL
              readOnlyRootFilesystem: false
              runAsNonRoot: true
              privileged: false
              runAsUser: 1000
              runAsGroup: 1000
            ports:
            - containerPort: 5055
            env:
            - name: LOG_LEVEL
              value: "info"
            - name: TZ
              value: "Etc/UTC"
            volumeMounts:
            - name: config
              mountPath: /app/config
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: seerr-longhorn
    ```
3. Add the following content to `seerr/seerr-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: seerr-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - seerr.example.com
        secretName: seerr-tls
      rules:
      - host: seerr.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: seerr-service
                port:
                  number: 5055
    ```
4. Add the following content to `seerr/seerr-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: seerr-longhorn
      namespace: arr-stack
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn
    ```
5. Add the following content to `seerr/seerr-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: seerr-service
      namespace: arr-stack
    spec:
      selector:
        app: seerr
      ports:
      - protocol: TCP
        port: 5055
        targetPort: 5055
    ```

6. Commit and push the changes to your Git repository.
