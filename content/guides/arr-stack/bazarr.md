+++
date = '2026-02-26T18:02:09+05:30'
draft = false
title = 'Bazarr'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Bazarr
Bazarr is an open-source subtitle management tool that helps you find and manage subtitles for your media library.

## Installation
1. Create the following directory structure for Bazarr:
   ```
    bazarr/
    ├── bazarr.yml
    ├── bazarr-ingress.yml
    ├── bazarr-pvc.yml
    └── bazarr-svc.yml
    ```

2. Add the following content to `bazarr/bazarr.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: bazarr
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: bazarr
      template:
        metadata:
          labels:
            app: bazarr
        spec:
          containers:
          - name: bazarr
            image: linuxserver/bazarr:1.5.5
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
            - name: tv
              mountPath: /tv
            - name: config
              mountPath: /config
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: bazarr-longhorn
          - name: tv
            nfs:
              server: <IP_ADDRESS>
              path: <PATH>
          - name: movies
            nfs:
              server: <IP_ADDRESS>
              path: <PATH>
    ```
3. Add the following content to `bazarr/bazarr-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: bazarr-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - bazarr.example.com
        secretName: bazarr-tls
      rules:
      - host: bazarr.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bazarr-service
                port:
                  number: 6767
    ```
4. Add the following content to `bazarr/bazarr-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: bazarr-longhorn
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

5. Add the following content to `bazarr/bazarr-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: bazarr-service
      namespace: arr-stack
    spec:
      selector:
        app: bazarr
      ports:
      - protocol: TCP
        port: 6767
        targetPort: 6767
    ```

6. Commit and push the changes to your Git repository.
