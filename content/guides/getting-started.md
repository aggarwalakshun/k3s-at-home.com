+++
date = '2026-02-26T07:40:07+05:30'
draft = false
title = 'Getting Started'
weight = 1
+++

# K3s Homelab

## Getting Started

- This guild will help you build a five node, dual stack (ipv4 + ipv6) k3s cluster. The cluster will be configured with external load balancing.

- This cluster will be built to be accessible from the outside world, completely over ipv6. This means that you can access your cluster from anywhere in the world, without the need for a VPN or port forwarding.

- I am using ipv6 because my ISP provides me with a dynamic public ipv6 address, but no public ipv4 address.

- This guide will assume these IPs and hostnames for the nodes.
  - Master Nodes
    ```
    kube-01 - 192.168.1.11
    kube-02 - 192.168.1.12
    kube-03 - 192.168.1.13
    ```
  - Worker Nodes
    ```
    kube-04 - 192.168.1.14
    kube-05 - 192.168.1.15
    ```
  - Load Balancer Nodes
    ```
    kube-lb-01 - 192.168.1.51
    kube-lb-02 - 192.168.1.52
    kube-lb-03 - 192.168.1.53
    ```

- The load balancers will use `192.168.1.125` as the virtual IP address that will be used to access the cluster.

- I'll use debian-13 for all nodes.

### Preparing Load Balancers

1. On `kube-lb-01`, `kube-lb-02`, and `kube-lb-03` install `haproxy` and `keepalived` packages.
    ``` bash
    sudo apt update && sudo apt upgrade -y && \
    sudo apt install haproxy keepalived
    ```

2. Add these two files on `kube-lb-01`, `kube-lb-02`, and `kube-lb-03`.
    ```
    /etc/haproxy/haproxy.cfg
    /etc/keepalived/keepalived.conf
    ```
    ``` /etc/keepalived/keepalived.conf
    # keepalived.conf
    global_defs {
      enable_script_security
      script_user root
    }

    vrrp_script chk_haproxy {
        script 'killall -0 haproxy'
        interval 2
    }

    vrrp_instance haproxy-vip {
        interface eth0              # edit this
        state MASTER
        priority 300                # 100 for kube-lb-01, 200 for kube-lb-02, 300 for kube-lb-03

        virtual_router_id 51

        virtual_ipaddress {
            192.168.1.125/24        # edit this
        }

        track_script {
            chk_haproxy
        }
    }
    ```
    ``` /etc/haproxy/haproxy.cfg
    # haproxy.cfg
      global
              log /dev/log    local0
              log /dev/log    local1 notice
              chroot /var/lib/haproxy
              stats socket /run/haproxy/admin.sock mode 660 level admin
              stats timeout 30s
              user haproxy
              group haproxy
              daemon

              ca-base /etc/ssl/certs
              crt-base /etc/ssl/private

              ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
              ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
              ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

      defaults
              log     global
              mode    http
              option  httplog
              option  dontlognull
              timeout connect 5000
              timeout client  50000
              timeout server  50000
              errorfile 400 /etc/haproxy/errors/400.http
              errorfile 403 /etc/haproxy/errors/403.http
              errorfile 408 /etc/haproxy/errors/408.http
              errorfile 500 /etc/haproxy/errors/500.http
              errorfile 502 /etc/haproxy/errors/502.http
              errorfile 503 /etc/haproxy/errors/503.http
              errorfile 504 /etc/haproxy/errors/504.http

      frontend k3s-frontend
          bind *:6443
          mode tcp
          option tcplog
          default_backend k3s-backend

      backend k3s-backend
          mode tcp
          option tcp-check
          balance roundrobin
          default-server inter 10s downinter 5s

          # Master Nodes
          server kube-01 192.168.1.101:6443 check         # edit this
          server kube-02 192.168.1.102:6443 check         # edit this
          server kube-03 192.168.1.103:6443 check         # edit this
      ```

  - Edit the `haproxy.cfg` file to point to the correct IP addresses of your master nodes. The `keepalived.conf` file should also be edited to point to the correct network interface and virtual IP address.

3. Enable and start `haproxy` and `keepalived`
    ``` bash
    sudo systemctl enable haproxy keepalived && \
    sudo systemctl restart haproxy keepalived
    ```

### Installing k3s on master nodes

1. Generate a random token on `kube-01` that will be used to join the other master nodes to the cluster.
    ``` bash
    head -c 16 /dev/urandom | base64
    ```

2. Install k3s on `kube-01` with the `--cluster-init` flag and the token generated in the previous step.
    ``` bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=<replace-with-your-token> sh -s - server \ 
    --cluster-init \
    --tls-san=192.168.1.125 \
    --disable=traefik \
    --disable=servicelb \
    --flannel-ipv6-masq \
    --flannel-iface=eth0 \
    --cluster-cidr=10.42.0.0/16,2001:db8:42::/56 \
    --service-cidr=10.43.0.0/16,2001:db8:43::/112
    ```
    - The `--tls-san` flag is used to add the virtual IP address to the TLS certificate. This is necessary because we will be accessing the cluster using the virtual IP address.

3. Get the full k3s token and store it somewhere. You will need this token to join the other master nodes and worker nodes to the cluster.
    ``` bash
    cat /var/lib/rancher/k3s/server/token
    ```

4. Install k3s on `kube-02` and `kube-03` using the token in step 3 and the `--server` flag pointing to the ip address of `kube-01`.
    ``` bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=<replace-with-your-token> sh -s - server \
    --server=https://192.168.1.11:6443 \
    --tls-san=192.168.1.125 \
    --disable=traefik \
    --disable=servicelb \
    --flannel-ipv6-masq \
    --flannel-iface=eth0 \
    --cluster-cidr=10.42.0.0/16,2001:db8:42::/56 \
    --service-cidr=10.43.0.0/16,2001:db8:43::/112
    ```

### Installing k3s on worker nodes

1. Install k3s on `kube-04` and `kube-05` using the token in step 3 and the `--server` flag pointing to the virtual IP address of the load balancers.
    ``` bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=<replace-with-your-token> sh -s - agent \
    --server=https://192.168.1.125:6443 \
    --flannel-iface=eth0
    ```

