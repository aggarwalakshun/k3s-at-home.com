+++
date = '2026-02-27T14:22:04+05:30'
draft = false
title = 'Pulse'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'monitoring'
+++

# Prerequisites
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Flux]({{% ref "guides/fluxcd.md" %}})

## Pulse
Pulse is a monitoring tool that provides real-time insights into the performance and health of your proxmox and/or k3s cluster.

## Installation
1. Create the following directory structure for Pulse:
    ```
    pulse/
    ├── agent.yml
    ├── ingress.yml
    ├── pvc.yml
    ├── rbac.yml
    ├── helmrelease.yml
    ├── helmrepository.yml
    └── secret.yml
    ```
2. Add the following content to the `agent.yml` file:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: pulse-agent
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: pulse-agent
      template:
        metadata:
          labels:
            app: pulse-agent
        spec:
          serviceAccountName: pulse-agent
          containers:
            - name: pulse-agent
              image: rcourtman/pulse:5.1.14
              command: ["/opt/pulse/bin/pulse-agent-linux-amd64"]
              args:
                - --enable-kubernetes
              env:
                - name: PULSE_URL
                  value: "http://pulse.monitoring.svc.cluster.local:7655"
                - name: PULSE_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: pulse-agent-token
                      key: token
                - name: PULSE_AGENT_ID
                  value: "k3s"
                - name: PULSE_ENABLE_HOST
                  value: "true"
                - name: PULSE_KUBE_INCLUDE_ALL_PODS
                  value: "true"
                - name: PULSE_KUBE_INCLUDE_ALL_DEPLOYMENTS
                  value: "true"
                - name: HOST_PROC
                  value: "/host/proc"
                - name: HOST_SYS
                  value: "/host/sys"
                - name: HOST_ETC
                  value: "/host/etc"
              volumeMounts:
                - name: host-proc
                  mountPath: /host/proc
                  readOnly: true
                - name: host-sys
                  mountPath: /host/sys
                  readOnly: true
                - name: host-root
                  mountPath: /host/root
                  readOnly: true
              securityContext:
                privileged: true
                readOnlyRootFilesystem: true
                allowPrivilegeEscalation: true
              resources:
                requests:
                  cpu: 50m
                  memory: 128Mi
                limits:
                  memory: 512Mi
          volumes:
            - name: host-proc
              hostPath:
                path: /proc
            - name: host-sys
              hostPath:
                path: /sys
            - name: host-root
              hostPath:
                path: /
          tolerations:
            - operator: Exists
    ```
3. Add the following content to the `ingress.yml` file:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: pulse-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
    spec:
      ingressClassName: traefik
      tls:
      - hosts:
        - pulse.example.com
        secretName: pulse-tls
      rules:
      - host: pulse.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pulse
                port:
                  number: 7655
    ```
4. Add the following content to the `pvc.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pulse-data
      namespace: monitoring
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: longhorn
    ```
5. Add the following content to the `rbac.yml` file:
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: pulse-agent
      namespace: monitoring
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: pulse-agent-read
    rules:
      - apiGroups: [""]
        resources: ["nodes", "pods"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["apps"]
        resources: ["deployments"]
        verbs: ["get", "list", "watch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: pulse-agent-read
    subjects:
      - kind: ServiceAccount
        name: pulse-agent
        namespace: monitoring
    roleRef:
      kind: ClusterRole
      name: pulse-agent-read
      apiGroup: rbac.authorization.k8s.io
    ```
6. Add the following content to the `helmrelease.yml` file:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: pulse
      namespace: monitoring
    spec:
      interval: 6h
      chart:
        spec:
          chart: pulse
          version: "5.1.14"
          sourceRef:
            kind: HelmRepository
            name: pulse
            namespace: flux-system
          interval: 6h
      values:
        persistence: 
          enabled: true
          existingClaim: pulse-longhorn
        strategy:
          type: Recreate
    ```
7. Add the following content to the `helmrepository.yml` file:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: pulse
      namespace: flux-system
    spec:
      interval: 6h
      url: https://rcourtman.github.io/Pulse
    ```
8. Add the following content to the `secret-tmp.yml` file:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: pulse-agent-token
      namespace: monitoring
    type: Opaque
    data:
      token: <base64-encoded-token>
    ```
9. Encrypt the `secret-tmp.yml` file using Sealed Secrets and save the output as `secret.yml`:
    ```bash
    kubeseal -o yaml < secret-tmp.yml > secret.yml
    rm secret-tmp.yml
    ```
10. Commit and push the file to your git repository and Flux will deploy Pulse to your cluster.
