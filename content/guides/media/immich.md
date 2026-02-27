+++
date = '2026-02-27T11:24:46+05:30'
draft = false
title = 'Immich'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'media'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- Optionally, you can setup [nvidia support]({{% ref "guides/gpu-operator/nvidia.md" %}}) or [intel gpu support]({{% ref "guides/gpu-operator/intel.md" %}}) for better performance of Immich's machine learning features.

## Immich
Immich is a self-hosted photo and video backup solution. It provides a secure and private way to store and manage your media files.

## Installation
1. Create the following directory structure for Immich:
   ```
    immich/
    ├── immich-db.yml
    ├── immich-ingress.yml
    ├── immich-ml.yml
    ├── immich-pvc.yml
    ├── immich-redis.yml
    ├── immich-secret.yml
    ├── immich-service.yml
    └── immich.yml
   ```
2. Add the following content to `immich/immich-db.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: immich-psql
      namespace: media
    spec:
      selector:
        matchLabels:
          app: immich-psql
      serviceName: immich-psql
      replicas: 1
      template:
        metadata:
          labels:
            app: immich-psql
        spec:
          initContainers:
          - name: cleanup
            image: busybox
            command: ['sh', '-c', 'rm -rf /var/lib/postgresql/lost+found']
            volumeMounts:
            - name: immich-db
              mountPath: /var/lib/postgresql/
          containers:
          - name: immich-psql
            image: ghcr.io/immich-app/postgres:18-vectorchord0.5.3-pgvector0.8.1
            ports:
            - containerPort: 5432
              name: postgres
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: immich-postgres-secret
                  key: password
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_DB
              value: "immich"
            - name: POSTGRES_INITDB_ARGS
              value: "--data-checksums"
            volumeMounts:
            - mountPath: /var/lib/postgresql
              name: immich-db
      volumeClaimTemplates:
      - metadata:
          name: immich-db
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
          storageClassName: longhorn
    ```
3. Add the following content to `immich/immich-ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: immich-ingress
      namespace: media
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - immich.example.com
        secretName: immich-tls
      rules:
      - host: immich.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: immich-service
                port:
                  number: 2283
    ```
4. Add the following content to `immich/immich-ml.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: immich-ml
      namespace: media
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: immich-ml
      template:
        metadata:
          labels:
            app: immich-ml
        spec:
          #runtimeClassName: nvidia                                            # For Nvidia GPUs only
          containers:
          - name: immich-machine-learning
            #image: ghcr.io/immich-app/immich-machine-learning:v2.5.6-cuda     # For NVIDIA GPUs, use the CUDA image
            #image: ghcr.io/immich-app/immich-machine-learning:v2.5.6-openvino # For Intel GPUs, use the OpenVINO image
            image: ghcr.io/immich-app/immich-machine-learning:v2.5.6           # For CPU-only, use the standard image
            ports:
            - containerPort: 3003
            env:
            - name: REDIS_HOSTNAME
              value: "immich-redis-service"
            #- name: NVIDIA_VISIBLE_DEVICES                                    # For NVIDIA GPUs only
            #  value: "all"                                                    # For NVIDIA GPUs only
            #- name: MACHINE_LEARNING_DEVICE_IDS                               # For NVIDIA GPUs only
            #  value: "0"                                                      # For NVIDIA GPUS only
            volumeMounts:
            - name: model-cache
              mountPath: /cache
            #- name: dri                                                       # For Intel GPUs only
            #  mountPath: /dev/dri                                             # For Intel GPUs only
            # For NVIDIA GPUs, uncomment the following lines to request GPU resources
            #resources:
            #  requests:
            #    nvidia.com/gpu: "1"
            #  limits:
            #    nvidia.com/gpu: "1"
            # For Intel GPUs, uncomment the following lines to request GPU resources
            #resources:
            #  limits:
            #    gpu.intel.com/i915: "1"
            #  requests:
            #    gpu.intel.com/i915: "1"
          volumes:
          - name: model-cache
            persistentVolumeClaim:
              claimName: immich-cache-longhorn
          #- name: dri                                                         # For Intel GPUs only
          #  hostPath:                                                         # For Intel GPUs only
          #    path: /dev/dri                                                  # For Intel GPUs only
    ```
5. Add the following content to `immich/immich-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: immich-cache-longhorn
      namespace: media
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 10Gi
      storageClassName: longhorn

    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: immich-pictures-pvc
      namespace: media
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: immich-pictures-pv
      resources:
        requests:
          storage: 100Gi
    ```
6. Add the following content to `immich/immich-redis.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: immich-redis
      namespace: media
    spec:
      selector:
        matchLabels:
          app: immich-redis
      serviceName: immich-redis
      replicas: 1
      template:
        metadata:
          labels:
            app: immich-redis
        spec:
          containers:
          - name: redis
            image: docker.io/valkey/valkey:8-bookworm@sha256:fea8b3e67b15729d4bb70589eb03367bab9ad1ee89c876f54327fc7c6e618571
            ports:
            - containerPort: 6379
              name: redis
    ```
7. Add the following content to `immich/immich-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: immich-postgres-secret
      namespace: media
    type: Opaque
    data:
      password: <base64-encoded-db-password>
    ```
8. Encrypt the `immich-postgres-secret` using Sealed Secrets and save the output to `immich/immich-secret.yml`:
    ```bash
    kubeseal -o yaml < immich/immich-secret-tmp.yml > immich/immich-secret.yml && \
    rm immich/immich-secret-tmp.yml
    ```
9. Add the following content to `immich/immich-service.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: immich-service
      namespace: media
    spec:
      selector:
        app: immich-app
      ports:
      - port: 2283
        targetPort: 2283

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: immich-machine-learning-service
      namespace: media
    spec:
      selector:
        app: immich-ml
      ports:
      - port: 3003
        targetPort: 3003

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: immich-psql
      namespace: media
    spec:
      selector:
        app: immich-psql
      ports:
        - name: postgres
          port: 5432
          targetPort: 5432
      clusterIP: None

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: immich-redis
      namespace: media
    spec:
      selector:
        app: immich-redis
      ports:
        - name: redis
          port: 6379
          targetPort: 6379
      clusterIP: None
    ```
10. Add the following content to `immich/immich.yml`:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: immich-app
      namespace: media
    spec:
      strategy:
        type: Recreate
      replicas: 1
      selector:
        matchLabels:
          app: immich-app
      template:
        metadata:
          labels:
            app: immich-app
        spec:
          containers:
          - name: immich-server
            image: ghcr.io/immich-app/immich-server:v2.5.6
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - |
                  pg_isready -h immich-psql.media.svc.cluster.local -U postgres -p 5432
              initialDelaySeconds: 10
              periodSeconds: 5
              failureThreshold: 5
            ports:
            - containerPort: 2283
            env:
            - name: REDIS_HOSTNAME
              value: "immich-redis.media.svc.cluster.local"
            - name: DB_USERNAME
              value: "postgres"
            - name: DB_DATABASE_NAME
              value: "immich"
            - name: DB_HOSTNAME
              value: "immich-psql.media.svc.cluster.local"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: immich-postgres-secret
                  key: password
            volumeMounts:
            - mountPath: /usr/src/app/upload
              name: pictures
          volumes:
          - name: pictures
            nfs:
              server: <nfs-server-ip>
              path: <nfs-export-path>
    ```
11. Commit and push the changes to your Git repository. Flux will automatically deploy Immich to your Kubernetes cluster.
