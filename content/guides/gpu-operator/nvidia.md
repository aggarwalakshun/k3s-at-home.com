+++
date = '2026-02-27T11:15:51+05:30'
draft = false
title = 'Nvidia'
[[cascade]]
  [cascade.params]
    [[cascade.params.sidebarmenus]]
      identifier = 'gpu-operator'
+++

# Prerequisites
- [Flux-CD]({{% ref "guides/fluxcd.md" %}})
- Execute these commands to install the NVIDIA drivers and container toolkit on the host machine. This will allow the GPU Operator to manage the NVIDIA GPUs on the cluster.
- Add the NVIDIA package repositories
  ```bash
  curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
      sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
      sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
  ```
  ```
  sudo sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
  ```
  - Update the package lists
  ```bash
  sudo apt-get update
  ```
  - Install packages
  ```bash
  sudo apt install nvidia-driver nvidia-container-toolkit nvidia-container-toolkit-base
  ```
  - Configure the container runtime to use the NVIDIA runtime.
  ```bash
  sudo nvidia-ctk runtime configure --runtime=containerd
  ```

## Installation
1. Create the following directory structure for NVIDIA GPU Operator:
    ```
    nvidia-gpu-operator/
    ├── gpu-operator.yml
    ├── gpu-operator-policy.yml
    └── gpu-operator-repo.yml
    ```
2. Add the following content to `nvidia-gpu-operator/gpu-operator.yml`:
    ```yaml
    ---
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: gpu-operator
      namespace: gpu-operator
    spec:
      interval: 6h
      chart:
        spec:
          chart: gpu-operator
          version: "v25.10.1"
          sourceRef:
            kind: HelmRepository
            name: nvidia
            namespace: flux-system
          interval: 6h
      install:
        createNamespace: true
      upgrade:
        remediation:
          remediateLastFailure: true
      values:
        driver:
          enabled: false
        toolkit:
          env:
          - name: CONTAINERD_SOCKET
            value: /run/k3s/containerd/containerd.sock
          - name: CONTAINERD_CONFIG
            value: /var/lib/rancher/k3s/agent/etc/containerd/config.toml
      ```
3. Add the following content to `nvidia-gpu-operator/gpu-operator-policy.yml`:
    ```yaml
    ---
    apiVersion: nvidia.com/v1
    kind: ClusterPolicy
    metadata:
      annotations:
        meta.helm.sh/release-name: gpu-operator
        meta.helm.sh/release-namespace: gpu-operator
      generation: 2
      labels:
        app.kubernetes.io/component: gpu-operator
        app.kubernetes.io/instance: gpu-operator
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: gpu-operator
        app.kubernetes.io/version: v25.3.2
        helm.sh/chart: gpu-operator-v25.3.2
        helm.toolkit.fluxcd.io/name: gpu-operator
        helm.toolkit.fluxcd.io/namespace: gpu-operator
      name: cluster-policy
    spec:
      ccManager:
        defaultMode: "off"
        enabled: false
        env: []
        image: k8s-cc-manager
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v0.1.1
      cdi:
        default: false
        enabled: false
      daemonsets:
        labels:
          app.kubernetes.io/managed-by: gpu-operator
          helm.sh/chart: gpu-operator-v25.3.2
        priorityClassName: system-node-critical
        rollingUpdate:
          maxUnavailable: "1"
        tolerations:
        - effect: NoSchedule
          key: nvidia.com/gpu
          operator: Exists
        updateStrategy: RollingUpdate
      dcgm:
        enabled: false
        image: dcgm
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: 4.2.3-1-ubuntu22.04
      dcgmExporter:
        enabled: true
        env:
        - name: DCGM_EXPORTER_LISTEN
          value: :9400
        - name: DCGM_EXPORTER_KUBERNETES
          value: "true"
        - name: DCGM_EXPORTER_COLLECTORS
          value: /etc/dcgm-exporter/dcp-metrics-included.csv
        image: dcgm-exporter
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/k8s
        serviceMonitor:
          additionalLabels: {}
          enabled: false
          honorLabels: false
          interval: 15s
          relabelings: []
        version: 4.2.3-4.1.3-ubuntu22.04
      devicePlugin:
        config:
          default: any
          name: time-slicing-config
        enabled: true
        env:
        - name: PASS_DEVICE_SPECS
          value: "true"
        - name: FAIL_ON_INIT_ERROR
          value: "true"
        - name: DEVICE_LIST_STRATEGY
          value: envvar
        - name: DEVICE_ID_STRATEGY
          value: uuid
        - name: NVIDIA_VISIBLE_DEVICES
          value: all
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: all
        image: k8s-device-plugin
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia
        version: v0.17.3
      driver:
        certConfig:
          name: ""
        enabled: false
        image: driver
        imagePullPolicy: IfNotPresent
        kernelModuleConfig:
          name: ""
        licensingConfig:
          configMapName: ""
          nlsEnabled: true
        manager:
          env:
          - name: ENABLE_GPU_POD_EVICTION
            value: "true"
          - name: ENABLE_AUTO_DRAIN
            value: "false"
          - name: DRAIN_USE_FORCE
            value: "false"
          - name: DRAIN_POD_SELECTOR_LABEL
            value: ""
          - name: DRAIN_TIMEOUT_SECONDS
            value: 0s
          - name: DRAIN_DELETE_EMPTYDIR_DATA
            value: "false"
          image: k8s-driver-manager
          imagePullPolicy: IfNotPresent
          repository: nvcr.io/nvidia/cloud-native
          version: v0.8.0
        rdma:
          enabled: false
          useHostMofed: false
        repoConfig:
          configMapName: ""
        repository: nvcr.io/nvidia
        startupProbe:
          failureThreshold: 120
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 60
        upgradePolicy:
          autoUpgrade: true
          drain:
            deleteEmptyDir: false
            enable: false
            force: false
            timeoutSeconds: 300
          maxParallelUpgrades: 1
          maxUnavailable: 25%
          podDeletion:
            deleteEmptyDir: false
            force: false
            timeoutSeconds: 300
          waitForCompletion:
            timeoutSeconds: 0
        useNvidiaDriverCRD: false
        usePrecompiled: false
        version: 570.148.08
        virtualTopology:
          config: ""
      gdrcopy:
        enabled: false
        image: gdrdrv
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v2.5
      gfd:
        enabled: true
        env:
        - name: GFD_SLEEP_INTERVAL
          value: 60s
        - name: GFD_FAIL_ON_INIT_ERROR
          value: "true"
        image: k8s-device-plugin
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia
        version: v0.17.3
      hostPaths:
        driverInstallDir: /run/nvidia/driver
        rootFS: /
      kataManager:
        config:
          artifactsDir: /opt/nvidia-gpu-operator/artifacts/runtimeclasses
          runtimeClasses:
          - artifacts:
              pullSecret: ""
              url: nvcr.io/nvidia/cloud-native/kata-gpu-artifacts:ubuntu22.04-535.54.03
            name: kata-nvidia-gpu
            nodeSelector: {}
          - artifacts:
              pullSecret: ""
              url: nvcr.io/nvidia/cloud-native/kata-gpu-artifacts:ubuntu22.04-535.86.10-snp
            name: kata-nvidia-gpu-snp
            nodeSelector:
              nvidia.com/cc.capable: "true"
        enabled: false
        image: k8s-kata-manager
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v0.2.3
      mig:
        strategy: single
      migManager:
        config:
          default: all-disabled
          name: default-mig-parted-config
        enabled: true
        env:
        - name: WITH_REBOOT
          value: "false"
        gpuClientsConfig:
          name: ""
        image: k8s-mig-manager
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v0.12.2-ubuntu20.04
      nodeStatusExporter:
        enabled: false
        image: gpu-operator-validator
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v25.3.2
      operator:
        defaultRuntime: docker
        initContainer:
          image: cuda
          imagePullPolicy: IfNotPresent
          repository: nvcr.io/nvidia
          version: 12.8.1-base-ubi9
        runtimeClass: nvidia
      psa:
        enabled: false
      sandboxDevicePlugin:
        enabled: true
        image: kubevirt-gpu-device-plugin
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia
        version: v1.3.1
      sandboxWorkloads:
        defaultWorkload: container
        enabled: false
      toolkit:
        enabled: true
        env:
        - name: CONTAINERD_SOCKET
          value: /run/k3s/containerd/containerd.sock
        - name: CONTAINERD_CONFIG
          value: /var/lib/rancher/k3s/agent/etc/containerd/config.toml
        image: container-toolkit
        imagePullPolicy: IfNotPresent
        installDir: /usr/local/nvidia
        repository: nvcr.io/nvidia/k8s
        version: v1.17.8-ubuntu20.04
      validator:
        image: gpu-operator-validator
        imagePullPolicy: IfNotPresent
        plugin:
          env:
          - name: WITH_WORKLOAD
            value: "false"
        repository: nvcr.io/nvidia/cloud-native
        version: v25.3.2
      vfioManager:
        driverManager:
          env:
          - name: ENABLE_GPU_POD_EVICTION
            value: "false"
          - name: ENABLE_AUTO_DRAIN
            value: "false"
          image: k8s-driver-manager
          imagePullPolicy: IfNotPresent
          repository: nvcr.io/nvidia/cloud-native
          version: v0.8.0
        enabled: true
        image: cuda
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia
        version: 12.8.1-base-ubi9
      vgpuDeviceManager:
        config:
          default: default
          name: ""
        enabled: true
        image: vgpu-device-manager
        imagePullPolicy: IfNotPresent
        repository: nvcr.io/nvidia/cloud-native
        version: v0.3.0
      vgpuManager:
        driverManager:
          env:
          - name: ENABLE_GPU_POD_EVICTION
            value: "false"
          - name: ENABLE_AUTO_DRAIN
            value: "false"
          image: k8s-driver-manager
          imagePullPolicy: IfNotPresent
          repository: nvcr.io/nvidia/cloud-native
          version: v0.8.0
        enabled: false
        image: vgpu-manager
        imagePullPolicy: IfNotPresent
    ```
4. Add the following content to `nvidia-gpu-operator/gpu-operator-repo.yml`:
    ```yaml
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: HelmRepository
    metadata:
      name: nvidia
      namespace: flux-system
    spec:
      interval: 6h
      url: https://helm.ngc.nvidia.com/nvidia
    ```
5. Commit and push these files to your Git repository. The Flux CD will automatically detect the changes and deploy the NVIDIA GPU Operator to your cluster.
