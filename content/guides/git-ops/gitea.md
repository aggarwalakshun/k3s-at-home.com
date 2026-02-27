+++
date = '2026-02-26T18:56:17+05:30'
draft = false
title = 'Gitea'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'gitops'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Gitea
Gitea is a self-hosted Git service that provides a simple and efficient way to manage your Git repositories. It is a lightweight and easy-to-use alternative to other Git hosting services.

## Installation
1. Create the following directory structure for Gitea:
   ```
    gitea/
    ├── gitea.yml
    ├── gitea-db.yml
    ├── gitea-db-secret.yml
    ├── gitea-ingress.yml
    ├── gitea-ingress-route.yml
    ├── gitea-pvc.yml
    └── gitea-svc.yml
    ```
2. Add the following content to `gitea/gitea.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gitea-app
      namespace: git-ops
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: gitea-app
      template:
        metadata:
          labels:
            app: gitea-app
        spec:
          containers:
          - name: gitea
            image: gitea/gitea:1.25.4
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - |
                  nc -z gitea-db.git-ops.svc.cluster.local 5432
              initialDelaySeconds: 5
              periodSeconds: 5
              failureThreshold: 3
            ports:
            - containerPort: 22
              name: ssh
            - containerPort: 3000
              name: http
            env:
            - name: USER_UID
              value: "1000"
            - name: USER_GID
              value: "1000"
            - name: GITEA__database__DB_TYPE
              value: "postgres"
            - name: GITEA__database__HOST
              value: "gitea-db.git-ops.svc.cluster.local:5432"
            - name: GITEA__database__NAME
              value: "gitea"
            - name: GITEA__database__USER
              value: "gitea"
            - name: GITEA__database__PASSWD
              valueFrom:
                secretKeyRef:
                  name: gitea-db-secret
                  key: password
            volumeMounts:
            - name: gitea-data
              mountPath: /data
            - name: localtime
              mountPath: /etc/localtime
          volumes:
          - name: localtime
            hostPath:
              path: /etc/localtime
              type: File
          - name: gitea-data
            persistentVolumeClaim:
              claimName: gitea-app-longhorn
    ```
3. Add the following content to `gitea/gitea-db.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: gitea-db
      namespace: git-ops
    spec:
      selector:
        matchLabels:
          app: gitea-db
      serviceName: gitea-db
      replicas: 1
      template:
        metadata:
          labels:
            app: gitea-db
        spec:
          initContainers:
          - name: init-cleanup
            image: busybox
            command: ["rm", "-rf", "/var/lib/postgresql/lost+found"]
            volumeMounts:
            - name: gitea-db
              mountPath: /var/lib/postgresql
          containers:
          - name: gitea-db
            image: postgres:18
            ports:
            - containerPort: 5432
            env:
            - name: POSTGRES_USER
              value: "gitea"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: gitea-db-secret
                  key: password
            - name: POSTGRES_DB
              value: "gitea"
            volumeMounts:
            - name: gitea-db
              mountPath: /var/lib/postgresql
      volumeClaimTemplates:
      - metadata:
          name: gitea-db
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
          storageClassName: longhorn
    ```
4. Add the following content to `gitea/gitea-db-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: gitea-db-secret
      namespace: git-ops
    type: Opaque
    data:
      password: <base64-encoded-password>
    ```
5. Encrypt the `gitea-db-secret-tmp.yml` file using Sealed Secrets and save the output as `gitea-db-secret.yml`:
    ```bash
    kubeseal -o yaml < gitea-db-secret-tmp.yml > gitea-db-secret.yml && \
    rm gitea-db-secret-tmp.yml
    ```
6. Add the following content to `gitea/gitea-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: gitea-ingress
      namespace: git-ops
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare 
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - gitea.example.com
        secretName: gitea-tls
      rules:
      - host: gitea.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitea-int-service
                port:
                  number: 3000
    ```
7. Add the following content to `gitea/gitea-ingress-route.yml`:
    ```yaml
    ---
    apiVersion: traefik.io/v1alpha1
    kind: IngressRouteTCP
    metadata:
      name: gitea-ssh
      namespace: git-ops
    spec:
      entryPoints:
        - ssh
      routes:
        - match: HostSNI(`*`)
          services:
            - name: gitea-int-service
              port: 22
    ```
8. Add the following content to `gitea/gitea-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: gitea-app-longhorn
      namespace: git-ops
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 2Gi
      storageClassName: longhorn
    ```
9. Add the following content to `gitea/gitea-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: gitea-int-service
      namespace: git-ops
    spec:
      selector:
        app: gitea-app
      ports:
        - protocol: TCP
          port: 3000
          targetPort: 3000
          name: http
        - protocol: TCP
          port: 22
          targetPort: 22
          name: ssh

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: gitea-db
      namespace: git-ops
    spec:
      ports:
        - port: 5432
          targetPort: 5432
      selector:
        app: gitea-db
      clusterIP: None
    ```
10. Commit and push the files to your Git repository. Flux will automatically deploy Gitea to your cluster.
