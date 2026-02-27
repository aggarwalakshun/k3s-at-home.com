+++
date = '2026-02-26T07:51:55+05:30'
draft = false
title = 'FluxCD'
Weight = 2
+++

# Prerequisites
FluxCD requires a working Kubernetes cluster. If you have not already set up a cluster, please refer to the [Getting-Started]({{% ref "guides/getting-started.md" %}}) guide.

## FluxCD
FluxCD is designed to manage and automate the deployment of applications in your Kubernetes cluster using GitOps principles.

## Installation
1. Create a Git repository to store your Flux configuration files. This repository will be used by Flux to sync the desired state of your cluster.

2. Install the flux-cli on your local machine:
    ``` bash
    curl -s https://fluxcd.io/install.sh | sudo bash
    ```
3. Bootstrap Flux in your Kubernetes cluster:
    ``` bash
    flux bootstrap git \
        --url=ssh://git@github.com/<your-username>/<your-repo>.git \
        --private-key-file=<path-to-your-private-key> \
        --branch=main \
        --path=clusters/default
    ```
    Replace `<your-username>`, `<your-repo>`, and `<path-to-your-private-key>` with your GitHub username, repository name, and the path to your SSH private key, respectively.

4. Verify the installation:
    ``` bash
    flux events
    ```
    This command will show you the events related to Flux and confirm that it is running correctly.
