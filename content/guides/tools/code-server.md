+++
date = '2026-02-27T14:42:51+05:30'
draft = false
title = 'Code Server'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Code Server
Code Server is an open-source project that allows you to run Visual Studio Code (VS Code) in a web browser.

## Installation
1. Create the following directory structure for Code Server:
   ```
    code-server/
    ├── ingress.yml
    ├── pvc.yml
    ├── secret.yml
    ├── svc.yml
    └── code-server.yml
   ```
2. Add the following content to `code-server/ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: code-server-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - vs.akshun-lab.cc
      secretName: code-server-tls
    rules:
    - host: vs.akshun-lab.cc
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: code-server-service
              port:
                number: 8443
  ```
3. Add the following content to `code-server/pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: code-server-longhorn
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
4. Add the following content to `code-server/secret-tmp.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: code-server-secrets
    namespace: tools
  type: Opaque
  data:
    PASSWORD: <base64-encoded-password>
    SUDO_PASSWORD: <base64-encoded-sudo-password>
  ```
5. Encrypt the `code-server/secret-tmp.yml` file using kubseal and save the output to `code-server/secret.yml`:
  ```bash
  kubeseal -o yaml < code-server/secret-tmp.yml > code-server/secret.yml && \
  rm code-server/secret-tmp.yml
  ```
6. Add the following content to `code-server/svc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: code-server-service
    namespace: tools
  spec:
    selector:
      app: code-server
    ports:
    - port: 8443
      targetPort: 8443
      protocol: TCP
  ```
7. Add the following content to `code-server/code-server.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: code-server
    namespace: tools
  spec:
    strategy:
      type: Recreate
    replicas: 1
    selector:
      matchLabels:
        app: code-server
    template:
      metadata:
        labels:
          app: code-server
      spec:
        containers:
        - name: code-server
          image: lscr.io/linuxserver/code-server:4.109.2
          ports:
          - containerPort: 8443
          env:
          - name: PUID
            value: "1000"
          - name: PGID
            value: "1000"
          - name: TZ
            value: "Etc/UTC"
          - name: PASSWORD
            valueFrom:
              secretKeyRef:
                name: code-server-secrets
                key: PASSWORD
          - name: SUDO_PASSWORD
            valueFrom:
              secretKeyRef:
                name: code-server-secrets
                key: SUDO_PASSWORD
          - name: DEFAULT_WORKSPACE
            value: "/config/workspace"
          volumeMounts:
          - name: code-server
            mountPath: /config
        volumes:
        - name: code-server
          persistentVolumeClaim:
            claimName: code-server-longhorn
  ```
8. Commit and push the changes to your Git repository and Flux will deploy code-server to your cluster.
