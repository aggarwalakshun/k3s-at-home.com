+++
date = '2026-02-27T12:20:00+05:30'
draft = false
title = 'Invidious'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'media'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Invidious
Invidious is an open-source alternative front-end to YouTube.

## Installation
1. Create the following directory structure for Invidious:
   ```
    invidious/
    ├── companion.yml
    ├── cm.yml
    ├── db.yml
    ├── db-secret.yml
    ├── ingress.yml
    ├── secret.yml
    ├── svc.yml
    └── invidious.yml
   ```
2. Add the following content to `companion.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: invidious-companion
      namespace: media
    spec:
      selector:
        matchLabels:
          app: invidious-companion
      template:
        metadata:
          labels:
            app: invidious-companion
        spec:
          containers:
          - name: inv-companion
            image: quay.io/invidious/invidious-companion:latest
            env:
            - name: SERVER_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: invidious-secrets
                  key: INVIDIOUS_COMPANION_KEY
            securityContext:
              capabilities:
                drop:
                - ALL
    ```
3. Add the following content to `cm.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: invidious-config
      namespace: media
    data:
      invidious.yml: |
        db:
          dbname: invidious
          user: kemal
          password: ${INVIDIOUS_DB_PASSWORD}
          host: invidious-db.media.svc.cluster.local
          port: 5432
        check_tables: true
        invidious_companion:
        - private_url: "http://invidious-companion-service.media.svc.cluster.local:8282/companion"
        invidious_companion_key: ${INVIDIOUS_COMPANION_KEY}
        hmac_key: ${INVIDIOUS_HMAC_KEY}
    ```
4. Add the following content to `db-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: invidious-secrets
      namespace: media
    type: Opaque
    data:
      postgres-db: <base64-encoded-db-name>
      postgres-user: <base64-encoded-db-user>
      postgres-password: <base64-encoded-db-password>
    ```
5. Encrypt the `db-secret-tmp.yml` file using Sealed Secrets and save the output as `db-secret.yml`:
    ```bash
    kubeseal -o yaml < db-secret-tmp.yml > db-secret.yml && \
    rm db-secret-tmp.yml
    ```
6. Add the following content to `db.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: invidious-db
      namespace: media
    spec:
      selector:
        matchLabels:
          app: invidious-db
      serviceName: invidious-db
      replicas: 1
      template:
        metadata:
          labels:
            app: invidious-db
        spec:
          initContainers:
          - name: clean-db-dir
            image: busybox
            command:
            - sh
            - -c
            - |
              rm -rf /var/lib/postgresql/lost+found
            volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql
          containers:
          - name: postgres
            image: postgres:18
            env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: invidious-db-secrets
                  key: postgres-db
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: invidious-db-secrets
                  key: postgres-user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: invidious-db-secrets
                  key: postgres-password
            volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql
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
7. Add the following content to `ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: invidious-ingress
      namespace: media
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - invidious.example.com
        secretName: invidious-tls
      rules:
      - host: invidious.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: invidious-service
                port:
                  number: 3000
    ```
8. Add the following content to `secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: invidious-secrets
      namespace: media
    type: Opaque
    data:
      INVIDIOUS_COMPANION_KEY: <base64-encoded-companion-key>
      INVIDIOUS_HMAC_KEY: <base64-encoded-hmac-key>
      INVIDIOUS_DB_PASSWORD: <base64-encoded-db-password>
    ```
9. Encrypt the `secret-tmp.yml` file using Sealed Secrets and save the output as `secret.yml`:
    ```bash
    kubeseal -o yaml < secret-tmp.yml > secret.yml && \
    rm secret-tmp.yml
    ```
10. Add the following content to `svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: invidious-service
      namespace: media
    spec:
      selector:
        app: invidious
      ports:
      - port: 3000
        targetPort: 3000
        protocol: TCP

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: invidious-companion-service
      namespace: media
    spec:
      selector:
        app: invidious-companion
      ports:
      - port: 8282
        targetPort: 8282

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: invidious-db
      namespace: media
    spec:
      selector:
        app: invidious-db
      ports:
      - port: 5432
        targetPort: 5432
      clusterIP: None
    ```
11. Add the following content to `invidious.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: invidious
      namespace: media
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: invidious
      template:
        metadata:
          labels:
            app: invidious
        spec:
          initContainers:
          - name: substitute-config
            image: alpine
            envFrom:
            - secretRef:
                name: invidious-secrets
            command:
            - sh
            - -c
            - apk add gettext && envsubst < /mnt/init/invidious.yml > /mnt/invidious.yml
            volumeMounts:
            - name: invidious-config
              mountPath: /mnt/init/invidious.yml
              subPath: invidious.yml
            - name: tmp
              mountPath: /mnt
              subPath: invidious.yml
          containers:
          - name: invidious
            image: quay.io/invidious/invidious@sha256:9d972ea5930c2e170b3c4d49bdd9fa09bf03f077d555f58747342062dffc5876
            command:
            - sh
            - -c
            - |
              export INVIDIOUS_CONFIG="$(cat /mnt/invidious.yml)" &&
              exec /invidious/invidious
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - |
                  nc -z invidious-db.media.svc.cluster.local 5432 && nc -z invidious-companion-service.media.svc.cluster.local 8282
            env:
            - name: INVIDIOUS_PORT
              value: "3000"
            ports:
            - containerPort: 3000
            volumeMounts:
            - name: logging
              mountPath: /var/log/invidious
            - name: tmp
              mountPath: /mnt
              subPath: invidious.yml
          volumes:
          - name: logging
            emptyDir: {}
          - name: tmp
            emptyDir: {}
          - name: invidious-config
            configMap:
              name: invidious-config
    ```
12. Commit and push the changes to your Git repository. FluxCD will automatically deploy Invidious to your Kubernetes cluster.
