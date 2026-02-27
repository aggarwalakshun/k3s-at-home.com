+++
date = '2026-02-27T14:49:29+05:30'
draft = false
title = 'Nextcloud'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Sealed Secrets]({{% ref "guides/sealed-secrets.md" %}})

## Nextcloud
Nextcloud is an open-source, self-hosted file sharing and collaboration platform. I will be deploying collabora online with nextcloud to enable online document editing.

## Installation
1. Create the following directory structure for Nextcloud:
  ```
    nextcloud/
    ├── collabora.yml
    ├── nextcloud-db.yml
    ├── nextcloud-ingress.yml
    ├── nextcloud-pvc.yml
    ├── nextcloud-secret.yml
    ├── nextcloud-svc.yml
    └── nextcloud.yml
  ```
2. Add the following content to `collabora.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: collabora
    namespace: tools
  spec:
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: collabora
    template:
      metadata:
        labels:
          app: collabora
      spec:
        containers:
        - name: collabora
          image: collabora/code:25.04.9.1.1
          ports:
          - containerPort: 9980
          env:
          - name: aliasgroup1
            value: https://nextcloud.example.com
          securityContext:
            capabilities:
              add:
                - MKNOD
  ```
3. Add the following content to `nextcloud-db.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: nextcloud-db
    namespace: tools
  spec:
    selector:
      matchLabels:
        app: nextcloud-db
    serviceName: nextcloud-db
    replicas: 1
    template:
      metadata:
        labels:
          app: nextcloud-db
      spec:
        containers:
        - name: nextcloud-db
          image: mariadb:12.2.2
          ports:
          - containerPort: 3306
          env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: nextcloud-secrets
                key: root-password
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: nextcloud-secrets
                key: user-password
          - name: MYSQL_DATABASE
            value: "nextcloud"
          - name: MYSQL_USER
            value: "nextcloud"
          - name: MARIADB_AUTO_UPGRADE
            value: "1"
          volumeMounts:
          - name: nextcloud-db
            mountPath: /var/lib/mysql
    volumeClaimTemplates:
    - metadata:
        name: nextcloud-db
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
        storageClassName: longhorn
  ```
4. Add the following content to `nextcloud-ingress.yml`:
  ```yaml
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: nextcloud-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - nextcloud.example.com
      secretName: nextcloud-tls
    rules:
    - host: nextcloud.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nextcloud-service
              port:
                number: 443

  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: collabora-ingress
    namespace: tools
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
  spec:
    ingressClassName: traefik
    tls:
    - hosts:
      - collabora.example.com
      secretName: collabora-tls
    rules:
    - host: collabora.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: collabora-service
              port:
                number: 9980
  ```
5. Add the following content to `nextcloud-pvc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nextcloud-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 2Gi
    storageClassName: longhorn

  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nextcloud-data-longhorn
    namespace: tools
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 10Gi
    storageClassName: longhorn
  ```
6. Add the following content to `nextcloud-secret-tmp.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: nextcloud-secrets
    namespace: tools
  type: Opaque
  data:
    root-password: <base64-encoded-root-password>
    user-password: <base64-encoded-user-password>
  ```
  - Encrypt the secret using kubeseal.
    ```bash
    kubeseal -o yaml < nextcloud-secret-tmp.yml > nextcloud-secret.yml && \
    rm nextcloud-secret-tmp.yml
    ```
7. Add the following content to `nextcloud-svc.yml`:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nextcloud-service
    namespace: tools
    annotations:
      traefik.ingress.kubernetes.io/service.serversscheme: https
      traefik.ingress.kubernetes.io/service.serverstransport: tools-insecure-transport@kubernetescrd
  spec:
    selector:
      app: nextcloud
    ports:
    - protocol: TCP
      port: 443
      targetPort: 443

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: collabora-service
    namespace: tools
    annotations:
      traefik.ingress.kubernetes.io/service.serversscheme: https
      traefik.ingress.kubernetes.io/service.serverstransport: tools-insecure-transport@kubernetescrd
  spec:
    selector:
      app: collabora
    ports:
    - protocol: TCP
      port: 9980
      targetPort: 9980

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nextcloud-db
    namespace: tools
  spec:
    selector:
      app: nextcloud-db
    ports:
      - protocol: TCP
        port: 3306
        targetPort: 3306
    clusterIP: None
  ```
  - For this service to work, you need to create an `insecureSkipVerify` server transport in Traefik. You can do this by creating the following CRD:
    ```yaml
    apiVersion: traefik.io/v1alpha1
    kind: ServersTransport
    metadata:
      name: insecure-transport
      namespace: tools
    spec:
      insecureSkipVerify: true
    ```
8. Add the following content to `nextcloud.yml`:
  ```yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nextcloud
    namespace: tools
  spec:
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: nextcloud
    template:
      metadata:
        labels:
          app: nextcloud
      spec:
        containers:
        - name: nextcloud
          image: lscr.io/linuxserver/nextcloud:33.0.0
          readinessProbe:
            exec:
              command:
              - sh
              - -c
              - nc -z nextcloud-db.tools.svc.cluster.local 3306
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          ports:
          - containerPort: 443
          env:
          - name: PGID
            value: "1000"
          - name: PUID
            value: "1000"
          - name: TZ
            value: "Etc/UTC"
          volumeMounts:
          - name: nextcloud-data
            mountPath: /data
          - name: nextcloud-config
            mountPath: /config
        volumes:
        - name: nextcloud-data
          persistentVolumeClaim:
            claimName: nextcloud-data-longhorn        
        - name: nextcloud-config
          persistentVolumeClaim:
            claimName: nextcloud-longhorn
  ```
9. Commit and push the files to your Git repository. Flux will automatically deploy the resources to your cluster.
