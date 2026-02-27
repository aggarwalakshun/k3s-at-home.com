+++
date = '2026-02-26T18:31:50+05:30'
draft = false
title = 'Sabnzbd'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})
- For NZB support, you will need an NZB indexer and Usenet provider. I recommend [AltHub](https://althub.co.za/) and[NewsHosting](https://newshosting.com/).

## Sabnzbd
Sabnzbd is an open-source binary newsreader that allows users to download files from Usenet servers.

## Installation
1. Create the following directory structure for Sabnzbd:
    ```
    sabnzbd/
    ├── sabnzbd.yml
    ├── sabnzbd-ingress.yml
    ├── sabnzbd-pvc.yml
    └── sabnzbd-svc.yml
    ```
2. Add the following content to `sabnzbd/sabnzbd.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sabnzbd
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app: sabnzbd
      template:
        metadata:
          labels:
            app: sabnzbd
        spec:
          containers:
          - name: sabnzbd
            image: lscr.io/linuxserver/sabnzbd:4.5.5
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
            volumeMounts:
            - name: sabnzbd-config
              mountPath: /config
            - name: downloads
              mountPath: /downloads
          volumes:
          - name: sabnzbd-config
            persistentVolumeClaim:
              claimName: sabnzbd-longhorn
          - name: downloads
            nfs:
              server: <IP_ADDRESS>
              path: <PATH_TO_DOWNLOADS>
    ```
3. Add the following content to `sabnzbd/sabnzbd-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: sabnzbd-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - sabnzbd.example.com
        secretName: sabnzbd-tls
      rules:
      - host: sabnzbd.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sabnzbd-service
                port:
                  number: 8080
    ```
4. Add the following content to `sabnzbd/sabnzbd-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: sabnzbd-longhorn
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
5. Add the following content to `sabnzbd/sabnzbd-svc.yml`
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: sabnzbd-service
      namespace: arr-stack
    spec:
      selector:
        app: sabnzbd
      ports:
        - protocol: TCP
          port: 8080
          targetPort: 8080
    ```
6. Commit and push the changes to your Git repository.
