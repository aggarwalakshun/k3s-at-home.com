+++
date = '2026-02-26T09:00:41+05:30'
draft = false
title = "Authelia"
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'tools'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})
- [Sealed-Secrets]({{% ref "guides/sealed-secrets.md" %}})
- [DDNS]({{% ref "guides/ddns.md" %}})
- [Cert-Manager]({{% ref "guides/cert-manager.md" %}})
- [Longhorn]({{% ref "guides/longhorn.md" %}})
- [Traefik]({{% ref "guides/traefik.md" %}})

## Authelia
Authelia is an open-source authentication and authorization server that provides a single sign-on (SSO) solution for your applications. It supports various authentication methods, including LDAP, TOTP, and WebAuthn, and can be easily integrated with reverse proxies like Traefik to secure access to your applications.

## Installation
1. Create the following directory structure for Authelia:
   ```
    authelia/
    ├── authelia-cm.yml
    ├── authelia-ingress.yml
    ├── authelia-middleware.yml
    ├── authelia-pvc.yml
    ├── authelia-release.yml
    ├── authelia-repository.yml
    └── authelia-secret.yml
   ```

2. Add the following content to `authelia/authelia-cm.yml`:
   ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: authelia-config
      namespace: tools
    data:
      configuration.yaml: |
        server:
          address: 'tcp4://:9091'
          buffers:
            read: 16384
        log:
          level: info 
          file_path: ''
          keep_stdout: true

        identity_validation:
          elevated_session:
            require_second_factor: true
          reset_password:
            jwt_lifespan: '5 minutes'

        theme: dark

        totp:
          disable: false
          issuer: 'example.com'
          period: 30
          skew: 1
          algorithm: 'sha1'
          digits: 6
          secret_size: 32
          allowed_algorithms:
            - 'SHA1'
          allowed_digits:
            - 6
          allowed_periods:
            - 30
          disable_reuse_security_policy: false

        password_policy:
          zxcvbn:
            enabled: true
            min_score: 4

        authentication_backend:
          file:
            path: '/config/users.yml'
            password:
              algorithm: 'argon2'
              argon2:
                variant: 'argon2id'
                iterations: 3
                memory: 65535
                parallelism: 4
                key_length: 32
                salt_length: 16

        access_control:
          default_policy: 'deny'
          rules:
            - domain: 'auth.example.com'
              policy: bypass
            - domain: 'invidious.example.com'
              resources: '^/(api/v1|feed|videoplayback|vi/.+\.(jpg|webp)|ggpht|latest_version|sb)'
              policy: bypass
            - domain: 'immich.example.com'
              policy: bypass
            - domain: 'jellyfin.example.com'
              policy: bypass
            - domain: 'gitea.example.com'
              policy: bypass
            - domain: 'nextcloud.example.com'
              policy: bypass
            - domain: 'collabora.example.com'
              policy: bypass
            - domain: 'vw.example.com'
              policy: bypass
            - domain: '*.example.com'
              policy: two_factor
        
        session:
          name: 'authelia_session'
          cookies:
            - domain: 'example.com'
              authelia_url: 'https://auth.example.com'

        regulation:
          max_retries: 4
          find_time: 120
          ban_time: 300

        storage:
          local:
            path: '/config/db.sqlite3'

        notifier:
          disable_startup_check: false
          smtp:
            address: submissions://smtp.gmail.com:465
            username: email@example.com
            sender: email@example.com
            identifier: localhost
            subject: "[Authelia] {title}"
            startup_check_address: email@example.com
            disable_require_tls: false
            disable_html_emails: false
            tls:
              skip_verify: false
              minimum_version: TLS1.2
        ntp:
          address: 'time.google.com:123'
          version: 4
          max_desync: '3s'
          disable_startup_check: false
    ```
3. Add the following content to `authelia/authelia-ingress.yml`:
   ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: authelia
      namespace: tools
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    spec:
      ingressClassName: traefik
      tls:
        - hosts:
            - auth.example.com
          secretName: authelia-tls
      rules:
        - host: auth.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: authelia
                    port:
                      number: 9091
    ```
4. Add the following content to `authelia/authelia-middleware.yml`:
    ```yaml
    apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: authelia
      namespace: tools
    spec:
      forwardAuth:
        address: http://authelia.tools.svc.cluster.local:9091/api/authz/forward-auth
        trustForwardHeader: true
        authResponseHeaders:
          - Remote-User
          - Remote-Groups
          - Remote-Name
          - Remote-Email
    ```
5. Add the following content to `authelia/authelia-pvc.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: authelia-pvc
      namespace: tools
    spec:
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
    ```
6. Add the following content to `authelia/authelia-release.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: authelia
      namespace: tools
    spec:
      interval: 6h
      chart:
        spec:
          chart: authelia
          version: "0.10.49"
          sourceRef:
            kind: HelmRepository
            name: authelia
            namespace: flux-system
          interval: 6h
      install:
        remediation:
          retries: 3
      upgrade:
        remediation:
          retries: 3
      values:
        configMap:
          notifier:
            smtp:
              enabled: true
              password:
                path: password
                secret_name: authelia-secrets
              username: email@example.com
          existingConfigMap: authelia-config
        persistence:
          enabled: true
          existingClaim: authelia-pvc
        secret:
          existingSecret: authelia-secrets
          additionalSecrets:
            authelia-secrets: {}
        pod:
          kind: Deployment
          strategy:
            type: Recreate
        service:
          port: 9091
      ```
7. Add the following content to `authelia/authelia-repository.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: authelia
      namespace: flux-system
    spec:
      interval: 6h
      url: https://charts.authelia.com
    ```
8. Add the following content to `authelia/authelia-secret-tmp.yml`:
    ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: authelia-secrets
      namespace: tools
    type: Opaque
    data:
      identity_validation.reset_password.jwt.hmac.key: <base64-encoded-secret-key>
      jwt.secret: <base64-encoded-secret-key>
      notifier.smtp.username: <base64-encoded-email-username>
      password: <base64-encoded-email-password>
      session.authentication.key: <base64-encoded-secret-key>
      session.encryption.key: <base64-encoded-secret-key>
      storage.encryption.key: <base64-encoded-secret-key>
    ```
9. Encrypt the `authelia/authelia-secret-tmp.yml` file using Sealed-Secrets and save the output as `authelia/authelia-secret.yml`:
    ```bash
    kubeseal --format yaml < authelia/authelia-secret-tmp.yml > authelia/authelia-secret.yml && \
    rm authelia/authelia-secret-tmp.yml
    ```
10. Commit and push the changes to your git repo and wait for fluxcd to deploy Authelia.
