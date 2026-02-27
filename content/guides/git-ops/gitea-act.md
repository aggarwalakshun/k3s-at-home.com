+++
date = '2026-02-27T10:24:04+05:30'
draft = false
title = 'Gitea-Act'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'gitops'
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [Gitea]({{% ref "guides/git-ops/gitea.md" %}})

## Gitea ACT
Gitea-Act is a tool that allows you to run Gitea Actions locally. It provides a way to test and debug your GitHub Actions workflows without needing to push changes to your repository.

## Installation
1. Create the following directory structure for Gitea-Act:
   ```
    gitea-act/
    ├── gitea-act.yml
    ├── gitea-act-secret.yml
    ├── gitea-act-docker-cm.yml
    └── gitea-act-runner-cm.yml
    ```
2. Create the `gitea-act.yml` file with the following content:
   ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: gitea-act-runner
      name: gitea-act-runner
      namespace: git-ops
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gitea-act-runner
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: gitea-act-runner
        spec:
          restartPolicy: Always
          volumes:
          - name: docker-certs
            emptyDir: {}
          - name: docker-ipv6
            configMap:
              name: docker-daemon-ipv6
          - name: runner-config
            configMap:
              name: gitea-act-runner-config
          containers:
          - name: runner
            image: gitea/act_runner:latest
            command: ["sh", "-c", "while ! nc -z localhost 2376 </dev/null; do echo 'waiting for docker daemon...'; sleep 5; done; /sbin/tini -- run.sh"]
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - |
                  nc -z gitea-int-service.git-ops.svc.cluster.local 3000
              initialDelaySeconds: 5
              periodSeconds: 5
              failureThreshold: 3
            env:
            - name: DOCKER_HOST
              value: tcp://localhost:2376
            - name: DOCKER_CERT_PATH
              value: /certs/client
            - name: DOCKER_TLS_VERIFY
              value: "1"
            - name: GITEA_INSTANCE_URL
              valueFrom:
                secretKeyRef:
                  key: URL
                  name: gitea-act-runner-secret
            - name: GITEA_RUNNER_REGISTRATION_TOKEN
              valueFrom:
                secretKeyRef:
                  key: TOKEN
                  name: gitea-act-runner-secret
            - name: CONFIG_FILE
              value: "/data/config.yaml"
            volumeMounts:
            - name: docker-certs
              mountPath: /certs
            - name: runner-data
              mountPath: /data/config.yaml
              subPath: config.yaml
          - name: daemon
            image: docker:29.2.1-dind
            env:
            - name: DOCKER_TLS_CERTDIR
              value: /certs
            securityContext:
              privileged: true
            volumeMounts:
            - name: docker-certs
              mountPath: /certs
            - name: docker-ipv6
              mountPath: /etc/docker/daemon.json
              subPath: daemon.json
    ```
3. Create the `gitea-act-secret-tmp.yml` file with the following content:
   ```yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: gitea-act-runner-secret
      namespace: git-ops
    type: Opaque
    data:
      URL: <base64-encoded-Gitea-instance-URL>
      TOKEN: <base64-encoded-runner-registration-token>
    ```
4. Encrypt the `gitea-act-secret-tmp.yml` file using Sealed Secrets and save it as `gitea-act-secret.yml`.
    ```bash
    kubeseal -o yaml < gitea-act-secret-tmp.yml > gitea-act-secret.yml && \
    rm gitea-act-secret-tmp.yml
    ```
5. Create the `gitea-act-docker-cm.yml` file with the following content:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: docker-daemon-ipv6
      namespace: git-ops
    data:
      daemon.json: |
        {
          "ipv6": true,
          "fixed-cidr-v6": "2001:db8:1::/64"
        }
    ```
6. Create the `gitea-act-runner-cm.yml` file with the following content:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: gitea-act-runner-config
      namespace: git-ops
    data:
      config.yaml: |
        #Sample configuration file, it's safe to copy this as the default config file without any modification.

        # You don't have to copy this file to your instance,
        # just run `./act_runner generate-config > config.yaml` to generate a config file.

        log:
          # The level of logging, can be trace, debug, info, warn, error, fatal
          level: info

        runner:
          # Where to store the registration result.
          file: .runner
          # Execute how many tasks concurrently at the same time.
          capacity: 5
          # Extra environment variables to run jobs.
          envs:
            A_TEST_ENV_NAME_1: a_test_env_value_1
            A_TEST_ENV_NAME_2: a_test_env_value_2
          # Extra environment variables to run jobs from a file.
          # It will be ignored if it's empty or the file doesn't exist.
          env_file: .env
          # The timeout for a job to be finished.
          # Please note that the Gitea instance also has a timeout (3h by default) for the job.
          # So the job could be stopped by the Gitea instance if it's timeout is shorter than this.
          timeout: 3h
          # The timeout for the runner to wait for running jobs to finish when shutting down.
          # Any running jobs that haven't finished after this timeout will be cancelled.
          shutdown_timeout: 0s
          # Whether skip verifying the TLS certificate of the Gitea instance.
          insecure: false
          # The timeout for fetching the job from the Gitea instance.
          fetch_timeout: 5s
          # The interval for fetching the job from the Gitea instance.
          fetch_interval: 2s
          # The github_mirror of a runner is used to specify the mirror address of the github that pulls the action repository.
          # It works when something like `uses: actions/checkout@v4` is used and DEFAULT_ACTIONS_URL is set to github,
          # and github_mirror is not empty. In this case,
          # it replaces https://github.com with the value here, which is useful for some special network environments.
          github_mirror: ''
          # The labels of a runner are used to determine which jobs the runner can run, and how to run them.
          # Like: "macos-arm64:host" or "ubuntu-latest:docker://docker.gitea.com/runner-images:ubuntu-latest"
          # Find more images provided by Gitea at https://gitea.com/docker.gitea.com/runner-images .
          # If it's empty when registering, it will ask for inputting labels.
          # If it's empty when execute `daemon`, will use labels in `.runner` file.
          labels:
            - "ubuntu-latest:docker://docker.gitea.com/runner-images:ubuntu-latest"
            - "ubuntu-22.04:docker://docker.gitea.com/runner-images:ubuntu-22.04"
            - "ubuntu-20.04:docker://docker.gitea.com/runner-images:ubuntu-20.04"

        cache:
          # Enable cache server to use actions/cache.
          enabled: true
          # The directory to store the cache data.
          # If it's empty, the cache data will be stored in $HOME/.cache/actcache.
          dir: ""
          # The host of the cache server.
          # It's not for the address to listen, but the address to connect from job containers.
          # So 0.0.0.0 is a bad choice, leave it empty to detect automatically.
          host: ""
          # The port of the cache server.
          # 0 means to use a random available port.
          port: 0
          # The external cache server URL. Valid only when enable is true.
          # If it's specified, act_runner will use this URL as the ACTIONS_CACHE_URL rather than start a server by itself.
          # The URL should generally end with "/".
          external_server: ""

        container:
          # Specifies the network to which the container will connect.
          # Could be host, bridge or the name of a custom network.
          # If it's empty, act_runner will create a network automatically.
          network: "bridge"
          # Whether to use privileged mode or not when launching task containers (privileged mode is required for Docker-in-Docker).
          privileged: false
          # And other options to be used when the container is started (eg, --add-host=my.gitea.url:host-gateway).
          options:
          # The parent directory of a job's working directory.
          # NOTE: There is no need to add the first '/' of the path as act_runner will add it automatically. 
          # If the path starts with '/', the '/' will be trimmed.
          # For example, if the parent directory is /path/to/my/dir, workdir_parent should be path/to/my/dir
          # If it's empty, /workspace will be used.
          workdir_parent:
          # Volumes (including bind mounts) can be mounted to containers. Glob syntax is supported, see https://github.com/gobwas/glob
          # You can specify multiple volumes. If the sequence is empty, no volumes can be mounted.
          # For example, if you only allow containers to mount the `data` volume and all the json files in `/src`, you should change the config to:
          # valid_volumes:
          #   - data
          #   - /src/*.json
          # If you want to allow any volume, please use the following configuration:
          # valid_volumes:
          #   - '**'
          valid_volumes: []
          # overrides the docker client host with the specified one.
          # If it's empty, act_runner will find an available docker host automatically.
          # If it's "-", act_runner will find an available docker host automatically, but the docker host won't be mounted to the job containers and service containers.
          # If it's not empty or "-", the specified docker host will be used. An error will be returned if it doesn't work.
          docker_host: ""
          # Pull docker image(s) even if already present
          force_pull: true
          # Rebuild docker image(s) even if already present
          force_rebuild: false
          # Always require a reachable docker daemon, even if not required by act_runner
          require_docker: false
          # Timeout to wait for the docker daemon to be reachable, if docker is required by require_docker or act_runner
          docker_timeout: 0s

        host:
          # The parent directory of a job's working directory.
          # If it's empty, $HOME/.cache/act/ will be used.
          workdir_parent:
    ```
7. Commit and push the files to your repository and flux will deploy the Gitea-Act runner to your cluster.
