+++
date = '2026-02-26T18:22:31+05:30'
draft = false
title = 'Qbittorrent'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Qbittorrent
Qbittorrent is a free and open-source BitTorrent client that allows users to download and manage torrent files.

## Installation
1. Create the following directory structure for Qbittorrent:
    ```
    qbittorrent/
    ├── qbittorrent.yml
    ├── qbittorrent-ingress.yml
    ├── qbittorrent-pvc.yml
    ├── qbittorrent-svc.yml
    ├── gluetun-secret.yml
    └── gluetun-cm.yml
    ```
2. Add the following content to `qbittorrent/qbittorrent.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: qbittorrent
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: qbittorrent
      template:
        metadata:
          labels:
            app: qbittorrent
        spec:
          initContainers:
          - name: gluetun
            image: qmcgaw/gluetun:v3.41.1
            restartPolicy: Always
            securityContext:
              capabilities:
                add:
                - NET_ADMIN
            envFrom:
            - configMapRef:
                name: gluetun-config
            env:
            - name: OPENVPN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: openvpn-secrets
                  key: OPENVPN_PASSWORD
            - name: OPENVPN_USER
              valueFrom:
                secretKeyRef:
                  name: openvpn-secrets
                  key: OPENVPN_USER
          containers:
          - name: qbittorrent
            image: linuxserver/qbittorrent:5.1.4
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
            volumeMounts:
            - name: downloads
              mountPath: /downloads
            - name: config
              mountPath: /config
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: qbittorrent-longhorn
          - name: downloads
            nfs:
              server: <IP_ADDRESS>
              path: <PATH_TO_NFS>
    ```
3. Add the following content to `qbittorrent/qbittorrent-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: qbittorrent-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - qbittorrent.example.com
        secretName: qbittorrent-tls
      rules:
      - host: qbittorrent.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: qbittorrent-service
                port:
                  number: 8080
    ```
4. Add the following content to `qbittorrent/qbittorrent-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: qbittorrent-longhorn
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
5. Add the following content to `qbittorrent/qbittorrent-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: qbittorrent-service
      namespace: arr-stack
    spec:
      selector:
        app: qbittorrent
      ports:
      - port: 8080
        targetPort: 8080
    ```
6. Add the following content to `qbittorrent/gluetun-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: openvpn-secrets
      namespace: arr-stack
    type: Opaque
    data:
      OPENVPN_USER: <BASE64_ENCODED_USERNAME>
      OPENVPN_PASSWORD: <BASE64_ENCODED_PASSWORD>
    ```
7. Encrypt the `gluetun-secret-tmp.yml` file using Sealed Secrets and save the output as `qbittorrent/gluetun-secret.yml`.
    ```bash
    kubeseal -o yaml < qbittorrent/gluetun-secret-tmp.yml > qbittorrent/gluetun-secret.yml && \
    rm qbittorrent/gluetun-secret-tmp.yml
    ```
8. Add the following content to `qbittorrent/gluetun-cm.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: gluetun-config
      namespace: arr-stack
    data:
      VPN_SERVICE_PROVIDER: "surfshark"
      SERVER_COUNTRIES: "Netherlands"
      HTTPPROXY: "ON"
      FIREWALL_OUTBOUND_SUBNETS: "192.168.1.0/24,10.42.0.0/16,10.43.0.0/16"
      DNS_ADDRESS: "8.8.8.8"
    ```
9. Commit the changes to your Git repository and push them to the remote repository.
