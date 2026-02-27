+++
date = '2026-02-27T14:30:10+05:30'
draft = false
title = 'Speedtest Tracker'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Sealed Secrets]({{% ref "guides/sealed-secrets.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Speedtest Tracker
Speedtest Tracker is a monitoring tool that allows you to track your internet speed.

## Installation
1. Create the following directory structure for Speedtest Tracker:
    ```
    speedtest-tracker/
    ├── ingress.yml
    ├── pvc.yml
    ├── secret.yml
    ├── svc.yml
    └── speedtest.yml
    ```
2. Add the following content to the `ingress.yml` file:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: speedtest-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - speedtest.example.com
        secretName: speedtest-tls
      rules:
      - host: speedtest.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: speedtest-service
                port:
                  number: 80
    ```
3. Add the following content to the `pvc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: speedtest-longhorn
      namespace: monitoring
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 100Mi
      storageClassName: longhorn
    ```
4. Add the following content to the `secret-tmp.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: speedtest-secret
      namespace: monitoring
    type: Opaque
    data:
      # echo -n 'base64:'; openssl rand -base64 32;
      app_key: <base64:app_key>
    ```
5. Encrypt the `secret-tmp.yml` file using kubeseal and save the output to `secret.yml`:
    ```bash
    kubeseal -o yaml < secret-tmp.yml > secret.yml
    rm secret-tmp.yml
    ```
6. Add the following content to the `svc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: speedtest-service
      namespace: monitoring
    spec:
      selector:
        app: speedtest
      ports:
      - port: 80
        targetPort: 80
        protocol: TCP
    ```
7. Add the following content to the `speedtest.yml` file:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: speedtest
      namespace: monitoring
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: speedtest
      template:
        metadata:
          labels:
            app: speedtest
        spec:
          containers:
          - name: speedtest
            image: lscr.io/linuxserver/speedtest-tracker:1.13.10
            ports:
            - containerPort: 80
            env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: "Etc/UTC"
            - name: DB_CONNECTION
              value: "sqlite"
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: speedtest-secret
                  key: app_key
            - name: SPEEDTEST_SCHEDULE
              value: "@hourly"
            - name: PRUNE_RESULTS_OLDER_THAN
              value: "7"
            - name: DISPLAY_TIMEZONE
              value: "Etc/UTC"
            - name: APP_TIMEZONE
              value: "Etc/UTC"
            volumeMounts:
            - name: config
              mountPath: /config
          volumes:
          - name: config
            persistentVolumeClaim:
              claimName: speedtest-longhorn
    ```
8. Commit and push the file to your git repository. Flux will automatically deploy the Speedtest Tracker to your cluster.
