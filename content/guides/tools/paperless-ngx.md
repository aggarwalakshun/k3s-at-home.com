+++
date = '2026-02-27T15:07:40+05:30'
draft = false
title = 'Paperless Ngx'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Paperless Ngx
Paperless Ngx is a document management system that allows you to digitize and organize your paper documents. This guide will also walk you through installing gotenberg and tika on your cluster.

## Installation
1. Create the following directory structure for Paperless Ngx:
  ```
    paperless-ngx/
    ├── db.yml
    ├── ingress.yml
    ├── svc.yml
    ├── pvc.yml
    ├── paperless-ngx.yml
    ├── gotenberg.yml
    └── tika.yml
  ```
2. Add the following content to `db.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: paperless-ngx-db
    namespace: tools
  spec:
    selector:
      matchLabels:
        app: paperless-ngx-db
    serviceName: paperless-ngx-db
    replicas: 1
    template:
      metadata:
        labels:
          app: paperless-ngx-db
      spec:
        containers:
        - name: paperless-ngx-db
          image: docker.io/library/redis:8
          ports:
          - containerPort: 6379
          volumeMounts:
          - name: paperless-ngx-db
            mountPath: /data
            subPath: redis
    volumeClaimTemplates:
    - metadata:
        name: paperless-ngx-db
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 500Mi
        storageClassName: longhorn
  ```
3. Add the following content to `ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: paperless-ngx-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - ngx.example.com
      secretName: paperless-ngx-tls
    rules:
    - host: ngx.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: paperless-ngx-service
              port:
                number: 8000
  ```
4. Add the following content to `svc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: paperless-ngx-service
    namespace: tools
  spec:
    selector:
      app: paperless-ngx
    ports:
    - port: 8000
      targetPort: 8000

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: paperless-ngx-db
    namespace: tools
  spec:
    selector:
      app: paperless-ngx-db
    ports:
    - port: 6379
      targetPort: 6379
    clusterIP: None

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: gotenberg-service
    namespace: tools
  spec:
    selector:
      app: gotenberg
    type: ClusterIP
    ports:
    - port: 3000
      targetPort: 3000

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: tika-service
    namespace: tools
  spec:
    type: ClusterIP
    selector:
      app: tika
    ports:
    - port: 9998
      targetPort: 9998
  ```
5. Add the following content to `pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: paperless-longhorn
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
6. Add the following content to `paperless-ngx.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: paperless-ngx
    namespace: tools
  spec:
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: paperless-ngx
    template:
      metadata:
        labels:
          app: paperless-ngx
      spec:
        containers:
        - name: paperless-ngx
          image: ghcr.io/paperless-ngx/paperless-ngx:2.20.8
          readinessProbe:
            exec:
              command:
              - bash
              - -c
              - |
                (echo >/dev/tcp/paperless-ngx-db.tools.svc.cluster.local/6379)
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          ports:
          - containerPort: 8000
          env:
          - name: PAPERLESS_REDIS
            value: "redis://paperless-ngx-db.tools.svc.cluster.local:6379"
          - name: PAPERLESS_URL
            value: "https://ngx.example.com"
          - name: PAPERLESS_TIME_ZONE
            value: "Etc/UTC"
          - name: PAPERLESS_TIKA_ENABLED
            value: "1"
          - name: PAPERLESS_TIKA_ENDPOINT
            value: "http://tika-service.tools.svc.cluster.local:9998"
          - name: PAPERLESS_TIKA_GOTENBERG_ENDPOINT
            value: "http://gotenberg-service.tools.svc.cluster.local:3000"
          volumeMounts:
          - name: data
            mountPath: /usr/src/paperless/data
            subPath: data
          - name: data
            mountPath: usr/src/paperless/media
            subPath: media
          - name: data
            mountPath: /usr/src/paperless/export
            subPath: export
          - name: data
            mountPath: /usr/src/paperless/consume
            subPath: consume
        volumes:
        - name: data
          persistentVolumeClaim:
            claimName: paperless-longhorn
  ```
7. Add the following content to `gotenberg.yml`:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: gotenberg
    namespace: tools
  spec:
    selector:
      matchLabels:
        app: gotenberg
    template:
      metadata:
        labels:
          app: gotenberg
      spec:
        securityContext:
          runAsUser: 1001
        containers:
        - name: gotenberg
          image: gotenberg/gotenberg:8.27
          command:
          - sh
          - -c
          - |
            gotenberg --chromium-disable-javascript=true --chromium-allow-list=file:///tmp/.*
          ports:
          - containerPort: 3000
          securityContext:
            readOnlyRootFilesystem: false
            allowPrivilegeEscalation: false
            privileged: false
  ```
8. Add the following content to `tika.yml`:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: tika
    namespace: tools
  spec:
    selector:
      matchLabels:
        app: tika
    template:
      metadata:
        labels:
          app: tika
      spec:
        containers:
        - name: tika
          image: apache/tika:3.2.3.0
          ports:
          - containerPort: 9998
  ```
9. Commit and push the changes to your Git repository. Flux will automatically deploy the resources to your cluster.
