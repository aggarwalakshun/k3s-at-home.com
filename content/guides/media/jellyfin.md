+++
date = '2026-02-27T12:32:43+05:30'
draft = false
title = 'Jellyfin'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'media'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})

## Jellyfin
Jellyfin is an open-source media server that allows you to organize, manage, and stream your media collection.

## Installation
1. Create the following directory structure for Jellyfin:
   ```
    jellyfin/
    ├── ingress.yml
    ├── pvc.yml
    ├── svc.yml
    └── jellyfin.yml
   ```
2. Add the following content to `ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: jellyfin-ingress
      namespace: media
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - jellyfin.example.com
        secretName: jellyfin-tls
      rules:
      - host: jellyfin.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jellyfin-service
                port:
                  number: 8096
    ```
3. Add the following content to `pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: jellyfin-pvc
      namespace: media
    spec:
      resources:
        requests:
          storage: 5Gi
      storageClassName: longhorn
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
    ```
4. Add the following content to `svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: jellyfin-service
      namespace: media
    spec:
      selector:
        app: jellyfin
      ports:
      - port: 8096
        targetPort: 8096
        protocol: TCP
    ```
5. Add the following content to `jellyfin.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jellyfin
      namespace: media
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: jellyfin
      template:
        metadata:
          labels:
            app: jellyfin
        spec:
          containers:
          - name: jellyfin
            image: jellyfin/jellyfin:10.11.6
            ports:
            - containerPort: 8096
            volumeMounts:
            - name: media
              mountPath: /media
            - name: config
              mountPath: /config
            - name: cache
              mountPath: /cache
            # Uncomment the following lines if you want to enable hardware acceleration for Intel GPUs
            #- name: i915
            #  mountPath: /dev/dri
            #securityContext:
            #    privileged: true
            #resources:
            #  requests:
            #    gpu.intel.com/i915: "1"
            #  limits:
            #    gpu.intel.com/i915: "1"
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: jellyfin-pvc
          - name: cache
            emptyDir: {}
          - name: media
            nfs:
              server: <nfs-server-ip>
              path: <path-to-media>
          # Uncomment the following lines if you want to enable hardware acceleration for Intel GPUs
          #- name: i915
          #  hostPath:
          #    path: /dev/dri
    ```
6. Commit and push the changes to your Git repository. FluxCD will automatically deploy Jellyfin to your Kubernetes cluster.
