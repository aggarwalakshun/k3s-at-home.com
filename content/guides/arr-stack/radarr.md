+++
date = '2026-02-26T18:28:03+05:30'
draft = false
title = 'Radarr'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Radarr
Radarr is an open-source movie collection manager that allows users to manage and organize their movie libraries.

## Installation
1. Create the following directory structure for Radarr:
   ```
    radarr/
    ├── radarr.yml
    ├── radarr-ingress.yml
    ├── radarr-pvc.yml
    └── radarr-svc.yml
    ```

2. Add the following content to `radarr/radarr.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: radarr
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: radarr
      template:
        metadata:
          labels:
            app: radarr
        spec:
          containers:
          - name: radarr
            image: lscr.io/linuxserver/radarr:6.0.4
            ports:
            - containerPort: 7878
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
            volumeMounts:
            - name: movies
              mountPath: /movies
            - name: downloads
              mountPath: /downloads
            - name: config
              mountPath: /config
          volumes:
          - name: movies
            nfs:
              server: <NFS_SERVER_IP>
              path: <PATH_TO_MOVIES>
          - name: downloads
            nfs:
              server: <NFS_SERVER_IP>
              path: <PATH_TO_DOWNLOADS>
          - name: config
            persistentVolumeClaim:
              claimName: radarr-longhorn
    ```
3. Add the following content to `radarr/radarr-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: radarr-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - radarr.example.com
        secretName: radarr-tls
      rules:
      - host: radarr.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: radarr-service
                port:
                  number: 7878
    ```
4. Add the following content to `radarr/radarr-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: radarr-longhorn
      namespace: arr-stack
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 2Gi
      storageClassName: longhorn
    ```
5. Add the following content to `radarr/radarr-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: radarr-service
      namespace: arr-stack
    spec:
      selector:
        app: radarr
      ports:
      - port: 7878
        targetPort: 7878
    ```
6. Commit and push the changes to your Git repository.
