+++
date = '2026-02-27T11:00:50+05:30'
draft = false
title = 'Semaphore'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'gitops'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Semaphore
Modern UI for Ansible, Terraform/OpenTofu/Terragrunt, PowerShell and other DevOps tools.

## Installation
1. Create the following directory structure for Semaphore:
    ```
    semaphore/
    ├── semaphore.yml
    ├── semaphore-db.yml
    ├── semaphore-secret.yml
    ├── semaphore-ingress.yml
    ├── semaphore-cm.yml
    └── semaphore-svc.yml
    ```
2. Add the following content to `semaphore/semaphore.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: semaphore
      namespace: git-ops
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: semaphore
      template:
        metadata:
          labels:
            app: semaphore
        spec:
          containers:
          - name: semaphore
            image: public.ecr.aws/semaphore/pro/server:v2.17.14
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - |
                  nc -z semaphore-db.git-ops.svc.cluster.local 3306
              initialDelaySeconds: 5
              periodSeconds: 5
              failureThreshold: 3
            ports:
            - name: http
              containerPort: 3000
            envFrom:
            - configMapRef:
                name: semaphore-config
            env:
            - name: SEMAPHORE_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: semaphore-secrets
                  key: admin_password
            - name: SEMAPHORE_DB_PASS
              valueFrom:
                secretKeyRef:
                  name: semaphore-secrets
                  key: mysql_password
            - name: SEMAPHORE_ACCESS_KEY_ENCRYPTION
              valueFrom:
                secretKeyRef:
                  name: semaphore-secrets
                  key: key
    ```
3. Add the following content to `semaphore/semaphore-db.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: semaphore-db
      namespace: git-ops
    spec:
      selector:
        matchLabels:
          app: semaphore-db
      serviceName: semaphore-db
      replicas: 1
      template:
        metadata:
          labels:
            app: semaphore-db
        spec:
          containers:
          - name: mysql
            image: mysql:9.6.0
            ports:
            - containerPort: 3306
            env:
            - name: MYSQL_RANDOM_ROOT_PASSWORD
              value: "'yes'"
            - name: MYSQL_DATABASE
              value: "semaphore"
            - name: MYSQL_USER
              value: "semaphore"
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: semaphore-secrets
                  key: mysql_password
            volumeMounts:
            - name: semaphore-db
              mountPath: /var/lib/mysql
      volumeClaimTemplates:
      - metadata:
          name: semaphore-db
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
          storageClassName: longhorn
    ```
4. Add the following content to `semaphore/semaphore-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: semaphore-secrets
      namespace: git-ops
    type: Opaque
    data:
      admin_password: <base64-encoded-random-string>
      mysql_password: <base64-encoded-random-string>
      key: <base64-encoded-random-string>
    ```
6. Encrypt the `semaphore/semaphore-secret-tmp.yml` file using Sealed Secrets and save it as `semaphore/semaphore-secret.yml`.
    ```bash
    kubeseal -o yaml < semaphore/semaphore-secret-tmp.yml > semaphore/semaphore-secret.yml && \
    rm semaphore/semaphore-secret-tmp.yml
    ```
7. Add the following content to `semaphore/semaphore-cm.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: semaphore-config
      namespace: git-ops
    data:
      SEMAPHORE_DB_USER: "semaphore"
      SEMAPHORE_DB_HOST: "semaphore-db"
      SEMAPHORE_DB_PORT: "3306"
      SEMAPHORE_DB_DIALECT: "mysql"
      SEMAPHORE_DB: "semaphore"
      SEMAPHORE_PLAYBOOK_PATH: "/tmp/semaphore"
      SEMAPHORE_ADMIN_NAME: "admin"
      SEMAPHORE_ADMIN_EMAIL: "admin@example.com"
      SEMAPHORE_ADMIN: "admin"
      SEMAPHORE_LDAP_ACTIVATED: "'no'"
    ```
8. Add the following content to `semaphore/semaphore-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: semaphore-ingress
      namespace: git-ops
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - semaphore.example.com
        secretName: semaphore-tls
      rules:
      - host: semaphore.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: semaphore-service
                port:
                  number: 3000
    ```
9. Add the following content to `semaphore/semaphore-svc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: semaphore-service
      namespace: git-ops
    spec:
      selector:
        app: semaphore
      ports:
      - name: http
        port: 3000
        targetPort: 3000

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: semaphore-db
      namespace: git-ops
    spec:
      selector:
        app: semaphore-db
      ports:
      - port: 3306
        targetPort: 3306
      clusterIP: None
    ```
10. Commit and push the files to your git repository. Flux will automatically deploy Semaphore to your Kubernetes cluster.
