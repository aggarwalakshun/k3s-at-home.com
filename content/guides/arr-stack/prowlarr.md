+++
date = '2026-02-26T18:16:00+05:30'
draft = false
title = 'Prowlarr'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'arr-stack'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Prowlarr
Prowlarr is an indexer manager/proxy built on the popular Radarr/Sonarr codebase. It integrates with your existing Sonarr/Radarr setup and allows you to manage all your indexers in one place.

## Installation
1. Create the following directory structure for Prowlarr:
   ```
    prowlarr/
    ├── prowlarr.yml
    ├── prowlarr-ingress.yml
    ├── prowlarr-pvc.yml
    ├── prowlarr-svc.yml
    ├── gluetun-secret.yml
    └── gluetun-cm.yml
    ```

2. Add the following content to `prowlarr/prowlarr.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: prowlarr
      namespace: arr-stack
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: prowlarr
      template:
        metadata:
          labels:
            app: prowlarr
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
          - name: prowlarr
            image: lscr.io/linuxserver/prowlarr:2.3.0
            volumeMounts:
            - name: config
              mountPath: /config
            ports:
            - containerPort: 9696
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: prowlarr-longhorn
    ```
3. Add the following content to `prowlarr/prowlarr-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: prowlarr-ingress
      namespace: arr-stack
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - prowlarr.example.com
        secretName: prowlarr-tls
      rules:
      - host: prowlarr.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prowlarr-service
                port:
                  number: 9696
    ```
4. Add the following content to `prowlarr/prowlarr-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: prowlarr-longhorn
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
5. Add the following content to `prowlarr/prowlarr-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: prowlarr-service
      namespace: arr-stack
    spec:
      selector:
        app: prowlarr
      ports:
        - protocol: TCP
          port: 9696
          targetPort: 9696
    ```
6. Add the following content to `prowlarr/gluetun-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: openvpn-secrets
      namespace: arr-stack
    type: Opaque
    data:
      OPENVPN_USER: <base64-encoded-openvpn-username>
      OPENVPN_PASSWORD: <base64-encoded-openvpn-password>
    ```
7. Encrypt the `gluetun-secret-tmp.yml` file using Sealed Secrets and save the output as `prowlarr/gluetun-secret.yml`:
    ```bash
    kubeseal -o yaml < prowlarr/gluetun-secret-tmp.yml > prowlarr/gluetun-secret.yml && \
    rm prowlarr/gluetun-secret-tmp.yml
    ```
8. Add the following content to `prowlarr/gluetun-cm.yml`:
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
9. Commit and push the changes to your Git repository.
