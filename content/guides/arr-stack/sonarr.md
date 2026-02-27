+++
date = '2026-02-26T18:36:37+05:30'
draft = false
title = 'Sonarr'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Sonarr
Sonarr is an open-source TV show management tool that allows users to automate the process of downloading and organizing TV shows for their media library.

## Installation
1. Create the following directory structure for Sonarr:
   ```
    sonarr/
    ├── sonarr.yml
    ├── sonarr-ingress.yml
    ├── sonarr-pvc.yml
    └── sonarr-svc.yml
    ```
2. Add the following content to `sonarr/sonarr.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sonarr
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: sonarr
      template:
        metadata:
          labels:
            app: sonarr
        spec:
          containers:
          - name: sonarr
            image: lscr.io/linuxserver/sonarr:4.0.16
            ports:
            - containerPort: 8989
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
            volumeMounts:
            - name: config
              mountPath: /config
            - name: tv
              mountPath: /tv
            - name: downloads
              mountPath: /downloads
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: sonarr-longhorn
          - name: downloads
            nfs:
              server: <IP_ADDRESS>
              path: <PATH>
          - name: tv
            nfs:
              server: <IP_ADDRESS>
              path: <PATH>
    ```
3. Add the following content to `sonarr/sonarr-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: sonarr-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - sonarr.example.com
        secretName: sonarr-tls
      rules:
      - host: sonarr.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sonarr-service
                port:
                  number: 8989
    ```
4. Add the following content to `sonarr/sonarr-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: sonarr-longhorn
      namespace: arr-stack
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 5Gi
      storageClassName: longhorn
    ```
5. Add the following content to `sonarr/sonarr-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: sonarr-service
      namespace: arr-stack
    spec:
      selector:
        app: sonarr
      ports:
      - port: 8989
        targetPort: 8989
    ```
6. Commit and push the files to your git repository.
