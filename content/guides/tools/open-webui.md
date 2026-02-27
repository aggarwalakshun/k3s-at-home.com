+++
date = '2026-02-27T15:04:08+05:30'
draft = false
title = 'Open Webui'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Ollama]({{% ref "guides/tools/ollama.md" %}})

## Open Webui
Open Webui is a web-based user interface for managing and interacting with large language models (LLMs) running on your infrastructure.

## Installation
1. Create the following directory structure for Open Webui:
  ```
    open-webui/
    ├── ingress.yml
    ├── pvc.yml
    ├── svc.yml
    └── open-webui.yml
  ```
2. Add the following content to `ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: open-webui-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - ollama.example.com
      secretName: open-webui-tls
    rules:
    - host: ollama.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: open-webui-service
              port:
                number: 8080
  ```
3. Add the following content to `pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: open-webui-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 2Gi
    storageClassName: longhorn
  ```
4. Add the following content to `svc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: open-webui-service
    namespace: tools
  spec:
    selector:
      app: open-webui
    ports:
    - port: 8080
      targetPort: 8080
  ```
5. Add the following content to `open-webui.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: open-webui
    namespace: tools
  spec:
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: open-webui
    template:
      metadata:
        labels:
          app: open-webui
      spec:
        containers:
        - name: open-webui
          image: ghcr.io/open-webui/open-webui:v0.8.5
          ports:
          - containerPort: 8080
          env:
          - name: OLLAMA_BASE_URL
            value: "http://ollama.tools.svc.cluster.local:11434"
          volumeMounts:
          - name: config
            mountPath: /app/backend/data
        volumes:
        - name: config
          persistentVolumeClaim:
            claimName: open-webui-longhorn
  ```
6. Commit and push the files to your Git repository. Flux will automatically deploy the resources to your cluster.
