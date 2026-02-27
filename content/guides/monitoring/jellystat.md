+++
date = '2026-02-27T13:34:24+05:30'
draft = false
title = 'Jellystat'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})
- [Jellyfin]({{% ref "guides/media/jellyfin.md" %}})

## Jellystat
Jellystat is a dashboard for monitoring the health and performance of jellyfin media server.

## Installation
1. Create the following directory structure for Jellystat:
    ```
    jellystat/
    ├── db.yml
    ├── ingress.yml
    ├── pvc.yml
    ├── secret.yml
    ├── jellystat.yml
    └── svc.yml
    ```
2. Add the following content to the `db.yml` file:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: jellystat-db
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: jellystat-db
      serviceName: jellystat-db
      replicas: 1
      template:
        metadata:
          labels:
            app: jellystat-db
        spec:
          containers:
          - name: jellystat-db
            image: postgres:18-alpine
            ports:
            - containerPort: 5432
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: jellystat-secret
                  key: password
            - name: POSTGRES_DB
              value: "jfstat"
            - name: POSTGRES_USER
              value: "postgres"
            - name: PGDATA
              value: /mnt/postgres/data
            volumeMounts:
            - name: postgres-data
              mountPath: /mnt/postgres
      volumeClaimTemplates:
      - metadata:
          name: postgres-data
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
          storageClassName: longhorn
    ```
3. Add the following content to the `ingress.yml` file:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: jellystat-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - jellystat.example.com
        secretName: jellystat-tls
      rules:
      - host: jellystat.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jellystat-service
                port:
                  number: 3000
    ```
4. Add the following content to the `pvc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: jellystat-backups-longhorn
      namespace: monitoring
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn
    ```
5. Add the following content to the `secret-tmp.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: jellystat-secret
      namespace: monitoring
    type: Opaque
    data:
      password: <base64-encoded-db-password>
      jwt: <base64-encoded-jwt-secret>
    ```
6. Encrypt the `secret-tmp.yml` file with kubeseal and save the output as `secret.yml`:
    ```bash
    kubeseal -o yaml < secret-tmp.yml > secret.yml && \
    rm secret-tmp.yml
    ```
7. Add the following content to the `jellystat.yml` file:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jellystat
      namespace: monitoring
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: jellystat
      template:
        metadata:
          labels:
            app: jellystat
        spec:
          containers:
          - name: jellystat
            image: cyfershepard/jellystat:1.1.8
            readinessProbe:
              exec:
                command:
                - bash
                - -c
                - |
                  (echo >/dev/tcp/jellystat-db.monitoring.svc.cluster.local/5432)
              initialDelaySeconds: 5
              periodSeconds: 5
              failureThreshold: 3
            env:
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: jellystat-secret
                  key: jwt
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: jellystat-secret
                  key: password
            - name: POSTGRES_IP
              value: "jellystat-db.monitoring.svc.cluster.local"
            - name: POSTGRES_PORT
              value: "5432"
            - name: POSTGRES_USER
              value: "postgres"
            volumeMounts:
            - name: backups
              mountPath: /app/backend/backup-data
          volumes:
          - name: backups
            persistentVolumeClaim:
              claimName: jellystat-backups-longhorn
    ```
8. Add the following content to the `svc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: jellystat-service
      namespace: monitoring
    spec:
      selector:
        app: jellystat
      ports:
      - port: 3000
        targetPort: 3000
        protocol: TCP

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: jellystat-db
      namespace: monitoring
    spec:
      selector:
        app: jellystat-db
      ports:
      - port: 5432
        targetPort: 5432
      clusterIP: None
    ```
9. Commit and push the changes to your git repository and flux will automatically deploy the application to your cluster.
