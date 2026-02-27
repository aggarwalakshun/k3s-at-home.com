+++
date = '2026-02-26T07:58:11+05:30'
draft = false
title = 'Cert Manager'
weight = 5
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})
- [Sealed-Secrets]({{% ref "guides/sealed-secrets.md" %}})
- [DDNS]({{% ref "guides/ddns.md" %}})
- [Traefik]({{% ref "guides/traefik.md" %}})

## Cert Manager
Cert Manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various issuing sources. It ensures that your applications can securely communicate over HTTPS by automatically renewing certificates before they expire.

## Installation
1. Create the following directory structure for Cert Manager:
   ```
    cert-manager/
    ├── cert-manager/
    │   ├── helmrelease.yml
    │   ├── helmrepository.yml
    │   ├── clusterissuer.yml
    │   └── cloudflare-token.yml
    └── namespace.yml
   ```

2. Add the following content to `cert-manager/namespace.yml`:
   ```yaml
   ---
    kind: Namespace
    apiVersion: v1
    metadata:
      name: cert-manager
      labels:
        name: cert-manager
    ```
3. Add the following content to `cert-manager/helmrepository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: jetstack
      namespace: flux-system
    spec:
      interval: 6h
      url: https://charts.jetstack.io
    ```
4. Add the following content to `cert-manager/helmrelease.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: cert-manager
      namespace: cert-manager
    spec:
      interval: 6h
      chart:
        spec:
          chart: cert-manager
          version: "v1.19.4"
          sourceRef:
            kind: HelmRepository
            name: jetstack
            namespace: flux-system
          interval: 6h
      install:
        remediation:
          retries: 3
      upgrade:
        remediation:
          retries: 3
      values:
        crds:
          enabled: true
          keep: true
    ```
5. Add the following content to `cert-manager/clusterissuer.yml`:
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-cloudflare
    spec:
      acme:
        email: email@example.com
        server: https://acme-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          name: letsencrypt-cloudflare
        solvers:
        - dns01:
            cloudflare:
              apiTokenSecretRef:
                name: cloudflare-api-token
                key: api-token
    ```
6. Add the following content to `cert-manager/cloudflare-token-tmp.yml`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-token
      namespace: cert-manager
    type: Opaque
    data:
      api-token: <base64-encoded-cloudflare-api-token>
    ```
7. Encrypt the `cloudflare-token-tmp.yml` file using Sealed-Secrets and save it as `cloudflare-token.yml`:
    ```bash
    kubeseal --format=yaml < cloudflare-token-tmp.yml > cloudflare-token.yml && \
    rm cloudflare-token-tmp.yml
    ```
8. Commit and push the changes to your Git repo.
