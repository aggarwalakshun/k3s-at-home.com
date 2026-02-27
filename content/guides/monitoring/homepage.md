+++
date = '2026-02-27T12:42:31+05:30'
draft = false
title = 'Homepage'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Authelia]({{% ref "guides/tools/authelia.md" %}})

## Homepage
A modern, fully static, fast, secure fully proxied, highly customizable application dashboard with integrations for over 100 services and translations into multiple languages. Easily configured via YAML files or through docker label discovery.

## Installation
1. Create the following directory structure for Homepage:
    ```
    homepage/
    ├── secret.yml
    ├── clusterole.yml
    ├── cm.yml
    ├── ingress.yml
    ├── pvc.yml
    ├── svc-account.yml
    ├── svc-account-token.yml
    ├── svc.yml
    └── homepage.yml
    ```
2. This secret will contain all api-keys and passwords for the services you want to integrate with Homepage. You can use the following template for your `secret-tmp.yml`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: homepage-secrets
      namespace: monitoring
    type: Opaque
    data:
      BAZARR_API_KEY: <base64-encoded-api-key>
      SONARR_API_KEY: <base64-encoded-password>
      # Add more services as needed
    ```
    - Encrypt the secret using Sealed Secrets like so:
      ```bash
      kubeseal -o yaml < secret-tmp.yml > secret.yml && \
      rm secret-tmp.yml
      ```
3. The `clusterole.yml` file will define the necessary permissions for Homepage to access the Kubernetes API. You can use the following template:
    ```yaml
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: homepage
      labels:
        app.kubernetes.io/name: homepage
    rules:
      - apiGroups:
          - ""
        resources:
          - namespaces
          - pods
          - nodes
        verbs:
          - get
          - list
      - apiGroups:
          - extensions
          - networking.k8s.io
        resources:
          - ingresses
        verbs:
          - get
          - list
      - apiGroups:
          - traefik.io
        resources:
          - ingressroutes
        verbs:
          - get
          - list
      - apiGroups:
          - gateway.networking.k8s.io
        resources:
          - httproutes
          - gateways
        verbs:
          - get
          - list
      - apiGroups:
          - metrics.k8s.io
        resources:
          - nodes
          - pods
        verbs:
          - get
          - list
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: homepage
      labels:
        app.kubernetes.io/name: homepage
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: homepage
    subjects:
      - kind: ServiceAccount
        name: homepage
        namespace: monitoring
    ```
4. The `cm.yml` file will contain the configuration for Homepage, including the services you want to integrate. This is an example configuration:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
        name: homepage
        namespace: monitoring
        labels:
            app.kubernetes.io/name: homepage
    data:
        docker.yaml: |
            ""
        bookmarks.yaml: |
            ""
        custom.css: |
            ""
        custom.js: |
            ""
        kubernetes.yaml: |
            mode: cluster
        widgets.yaml: |
            - kubernetes:
                  cluster:
                      show: true
                      cpu: true
                      memory: true
                      showLabel: true
                      label: "k3s-cluster"
                  nodes:
                      show: true
            - longhorn:
                  expanded: true
                  total: true
                  labels: true
                  nodes: false
        settings.yaml: |
            providers:
                longhorn:
                    url: http://longhorn-frontend.longhorn-system.svc.cluster.local:80
        services.yaml: |
            - Apps:
                  - Sonarr:
                        href: https://sonarr.example.com
                        description: series
                        icon: sonarr.png
                        namespace: arr-stack
                        podSelector: app=sonarr
                        app: sonarr
                        widget:
                            type: sonarr
                            url: http://sonarr-service.arr-stack.svc.cluster.local:8989
                            key: "${SONARR_API_KEY}"
                  - Bazarr:
                        href: https://bazarr.example.com
                        description: subtitles
                        icon: bazarr.png
                        namespace: arr-stack
                        podSelector: app=bazarr
                        app: bazarr
                        widget:
                            type: bazarr
                            url: http://bazarr-service.arr-stack.svc.cluster.local:6767
                            key: "${BAZARR_API_KEY}"
    ```
5. Add the following to `ingress.yml`:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: homepage-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.middlewares: tools-authelia@kubernetescrd
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - home.example.com
        secretName: homepage-tls
      rules:
      - host: home.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homepage-service
                port:
                  number: 3000
    ```
6. The `pvc.yml` file will define the persistent volume claim for Homepage images.
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: homepage-longhorn
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
7. The `svc-account.yml` file will create a service account for Homepage:
    ```yaml
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: homepage
      namespace: monitoring
      labels:
        app.kubernetes.io/name: homepage
    secrets:
      - name: homepage
    ```
8. The `svc-account-token.yml` file will create a secret for the service account token:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    type: kubernetes.io/service-account-token
    metadata:
      name: homepage
      namespace: monitoring
      labels:
        app.kubernetes.io/name: homepage
      annotations:
        kubernetes.io/service-account.name: homepage
9. Create a `svc.yml` file to define the service for Homepage:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: homepage-service
      namespace: monitoring
    spec:
      selector:
        app.kubernetes.io/name: homepage
      ports:
      - port: 3000
        targetPort: 3000
    ```
10. Finally, the `homepage.yml` file will define the deployment for Homepage. This deployment will substitute the api-keys from the secret into the configmap.
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: homepage
      namespace: monitoring
      labels:
        app.kubernetes.io/name: homepage
    spec:
      revisionHistoryLimit: 3
      replicas: 1
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app.kubernetes.io/name: homepage
      template:
        metadata:
          labels:
            app.kubernetes.io/name: homepage
        spec:
          serviceAccountName: homepage
          automountServiceAccountToken: true
          enableServiceLinks: true
          initContainers:
            - name: substitute-config
              image: alpine
              envFrom:
                - secretRef:
                    name: homepage-secrets
              command:
                - "sh"
                - "-c"
                - apk add gettext && envsubst < /mnt/init/services.yaml > /mnt/services.yaml
              volumeMounts:
                - name: homepage-config
                  mountPath: /mnt/init/services.yaml
                  subPath: services.yaml
                - name: tmp
                  mountPath: /mnt
                  subPath: services.yaml
          containers:
            - name: homepage
              image: "ghcr.io/gethomepage/homepage:v1.10.1"
              imagePullPolicy: IfNotPresent
              env:
                - name: HOMEPAGE_ALLOWED_HOSTS
                  value: "home.example.com"
              ports:
                - name: http
                  containerPort: 3000
                  protocol: TCP
              volumeMounts:
                - mountPath: /app/config/custom.js
                  name: homepage-config
                  subPath: custom.js
                - mountPath: /app/config/custom.css
                  name: homepage-config
                  subPath: custom.css
                - mountPath: /app/config/bookmarks.yaml
                  name: homepage-config
                  subPath: bookmarks.yaml
                - mountPath: /app/config/docker.yaml
                  name: homepage-config
                  subPath: docker.yaml
                - mountPath: /app/config/kubernetes.yaml
                  name: homepage-config
                  subPath: kubernetes.yaml
                - mountPath: /app/config
                  name: tmp
                  subPath: services.yaml
                - mountPath: /app/config/settings.yaml
                  name: homepage-config
                  subPath: settings.yaml
                - mountPath: /app/config/widgets.yaml
                  name: homepage-config
                  subPath: widgets.yaml
                - mountPath: /app/config/logs
                  name: logs
                - mountPath: /app/public/images
                  name: images
          volumes:
            - name: homepage-config
              configMap:
                name: homepage
            - name: images
              persistentVolumeClaim:
                claimName: homepage-longhorn
            - name: logs
              emptyDir: {}
            - name: tmp
              emptyDir: {}
    ```
11. Commit and push all the files to your repository. Flux will automatically deploy Homepage to your cluster.
