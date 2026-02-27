+++
date = '2026-02-26T08:22:41+05:30'
draft = false
title = 'DDNS with Cloudflare'
weight = 4
+++

# Prerequisites
This guide assumes you have the following prerequisites in place:
- [FluxCD]({{% ref "guides/fluxcd.md" %}})
- [Cert-Manager]({{% ref "guides/cert-manager.md" %}})
- A domain name registered with cloudflare.
- A docker registry to store the ddns-updater image.

## DDNS
In this guide, we will set up Dynamic DNS (DDNS) using Cloudflare as the DNS provider. This will allow us to automatically update our DNS records when our IP address changes, ensuring that our services remain accessible.

## Building the DDNS Updater Image
1. Create these two python files.
    - `ddns_updater.py`
      ```python
      import os
      import sys
      import time
      import logging
      import ipaddress

      import cloudflare
      from cloudflare import Cloudflare
      from kubernetes import client, config

      logging.basicConfig(
          level=logging.INFO,
          format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
          datefmt="%Y-%m-%dT%H:%M:%S",
          stream=sys.stdout,
      )
      logger = logging.getLogger("ddns-updater")

      CF_API_TOKEN: str = os.environ["CF_API_TOKEN"]
      CF_ZONE_ID: str = os.environ["CF_ZONE_ID"]
      DNS_DOMAIN: str = os.environ["DNS_DOMAIN"]
      DNS_TTL: int = int(os.getenv("DNS_TTL", "120"))
      DNS_PROXIED: bool = os.getenv("DNS_PROXIED", "false").lower() == "true"
      POLL_INTERVAL: int = int(os.getenv("POLL_INTERVAL", "300"))
      NAMESPACE: str = os.getenv("NAMESPACE", "ddns-updater")
      CONFIGMAP_NAME: str = os.getenv("CONFIGMAP_NAME", "node-ipv6-addresses")
      NODE_NAMES: list[str] = [
          n.strip() for n in os.environ["NODE_NAMES"].split(",") if n.strip()
      ]

      def load_k8s_config() -> None:
          try:
              config.load_incluster_config()
              logger.info("Loaded in-cluster Kubernetes config")
          except config.ConfigException:
              config.load_kube_config()
              logger.info("Loaded local kubeconfig")


      def read_node_ips(v1: client.CoreV1Api) -> dict[str, str]:
          try:
              cm = v1.read_namespaced_config_map(name=CONFIGMAP_NAME, namespace=NAMESPACE)
              data = cm.data or {}
          except client.exceptions.ApiException as exc:
              if exc.status == 404:
                  logger.warning("ConfigMap %s not found yet -- agents may not have started", CONFIGMAP_NAME)
                  return {}
              raise

          valid: dict[str, str] = {}
          for node, raw_ip in data.items():
              try:
                  ip = ipaddress.ip_address(raw_ip.strip())
                  if isinstance(ip, ipaddress.IPv6Address) and ip.is_global:
                      valid[node] = str(ip)
                  else:
                      logger.warning("Ignoring non-global IPv6 for node %s: %s", node, raw_ip)
              except ValueError:
                  logger.warning("Invalid IP in ConfigMap for node %s: %r", node, raw_ip)

          return valid

      def get_wildcard_records(cf: Cloudflare) -> dict[str, str]:
          wildcard = f"*.{DNS_DOMAIN}"
          records: dict[str, str] = {}
          page = 1
          while True:
              result = cf.dns.records.list(
                  zone_id=CF_ZONE_ID,
                  type="AAAA",
                  name=wildcard,
                  per_page=100,
                  page=page,
              )
              for rec in result.result:
                  records[rec.id] = rec.content
              if page >= result.result_info.total_pages:
                  break
              page += 1
          return records


      def sync_wildcard_records(cf: Cloudflare, desired_ips: list[str]) -> None:
          wildcard = f"*.{DNS_DOMAIN}"
          existing = get_wildcard_records(cf)
          existing_by_ip = {ip: rec_id for rec_id, ip in existing.items()}

          desired_set = set(desired_ips)
          existing_set = set(existing_by_ip.keys())

          to_delete = existing_set - desired_set
          to_create = desired_set - existing_set

          for ip in to_delete:
              rec_id = existing_by_ip[ip]
              logger.info("Deleting  %s -> %s", wildcard, ip)
              cf.dns.records.delete(dns_record_id=rec_id, zone_id=CF_ZONE_ID)

          for ip in to_create:
              logger.info("Creating  %s -> %s", wildcard, ip)
              cf.dns.records.create(
                  zone_id=CF_ZONE_ID,
                  type="AAAA",
                  name=wildcard,
                  content=ip,
                  ttl=DNS_TTL,
                  proxied=DNS_PROXIED,
              )

          if not to_delete and not to_create:
              logger.info("No changes needed for %s (%d records)", wildcard, len(desired_set))
          else:
              logger.info(
                  "Sync complete for %s: created=%d deleted=%d total=%d",
                  wildcard, len(to_create), len(to_delete), len(desired_set),
              )

      def run_once(cf: Cloudflare, v1: client.CoreV1Api) -> None:
          node_ips = read_node_ips(v1)

          if not node_ips:
              logger.warning("No node IPs available yet, skipping Cloudflare update")
              return

          missing = [n for n in NODE_NAMES if n not in node_ips]
          if missing:
              logger.warning("No IP reported yet for nodes: %s", missing)

          desired_ips = [node_ips[n] for n in NODE_NAMES if n in node_ips]
          logger.info("Round-robin IPs for *.%s: %s", DNS_DOMAIN, desired_ips)

          try:
              sync_wildcard_records(cf, desired_ips)
          except cloudflare.APIError as exc:
              logger.error("Cloudflare API error: %s", exc)


      def main() -> None:
          load_k8s_config()
          v1 = client.CoreV1Api()
          cf = Cloudflare(api_token=CF_API_TOKEN)

          logger.info(
              "DDNS updater started | zone=%s domain=%s nodes=%s poll=%ds",
              CF_ZONE_ID, DNS_DOMAIN, NODE_NAMES, POLL_INTERVAL,
          )

          while True:
              try:
                  run_once(cf, v1)
              except Exception as exc:
                  logger.exception("Unhandled error: %s", exc)
              logger.info("Sleeping %ds...", POLL_INTERVAL)
              time.sleep(POLL_INTERVAL)


      if __name__ == "__main__":
          main()
      ```
    - `node_agent.py`
      ```python
      import os
      import sys
      import time
      import logging
      import subprocess
      import ipaddress

      from kubernetes import client, config

      logging.basicConfig(
          level=logging.INFO,
          format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
          datefmt="%Y-%m-%dT%H:%M:%S",
          stream=sys.stdout,
      )
      logger = logging.getLogger("node-agent")

      NODE_NAME: str = os.environ["NODE_NAME"]
      NAMESPACE: str = os.getenv("NAMESPACE", "ddns-updater")
      CONFIGMAP_NAME: str = os.getenv("CONFIGMAP_NAME", "node-ipv6-addresses")
      POLL_INTERVAL: int = int(os.getenv("POLL_INTERVAL", "300"))

      IPV6_ECHO_URLS: list[str] = [
          "https://api6.ipify.org",
          "https://ipv6.icanhazip.com",
          "https://v6.ident.me",
      ]

      CURL_TIMEOUT: int = int(os.getenv("CURL_TIMEOUT", "10"))


      def fetch_ipv6() -> str | None:
          for url in IPV6_ECHO_URLS:
              try:
                  result = subprocess.run(
                      [
                          "curl",
                          "--silent",
                          "--fail",
                          "--max-time", str(CURL_TIMEOUT),
                          url,
                      ],
                      capture_output=True,
                      text=True,
                      timeout=CURL_TIMEOUT + 2,
                  )
                  if result.returncode != 0:
                      logger.warning("curl failed for %s: %s", url, result.stderr.strip())
                      continue

                  raw = result.stdout.strip()
                  ip = ipaddress.ip_address(raw)
                  if isinstance(ip, ipaddress.IPv6Address) and ip.is_global:
                      logger.info("Discovered IPv6 via %s: %s", url, raw)
                      return raw
                  logger.warning("Address from %s is not a global IPv6: %s", url, raw)

              except subprocess.TimeoutExpired:
                  logger.warning("curl timed out for %s", url)
              except ValueError:
                  logger.warning("Invalid IP returned from %s: %r", url, result.stdout.strip())
              except Exception as exc:
                  logger.warning("Unexpected error querying %s: %s", url, exc)

          return None


      def patch_configmap(v1: client.CoreV1Api, ipv6: str) -> None:
          patch_body = {
              "apiVersion": "v1",
              "kind": "ConfigMap",
              "metadata": {"name": CONFIGMAP_NAME, "namespace": NAMESPACE},
              "data": {NODE_NAME: ipv6},
          }
          try:
              v1.patch_namespaced_config_map(
                  name=CONFIGMAP_NAME,
                  namespace=NAMESPACE,
                  body=patch_body,
              )
              logger.info("Patched ConfigMap %s[%s] = %s", CONFIGMAP_NAME, NODE_NAME, ipv6)
          except client.exceptions.ApiException as exc:
              if exc.status == 404:
                  v1.create_namespaced_config_map(
                      namespace=NAMESPACE,
                      body=client.V1ConfigMap(
                          metadata=client.V1ObjectMeta(
                              name=CONFIGMAP_NAME,
                              namespace=NAMESPACE,
                          ),
                          data={NODE_NAME: ipv6},
                      ),
                  )
                  logger.info("Created ConfigMap %s with %s = %s", CONFIGMAP_NAME, NODE_NAME, ipv6)
              else:
                  raise


      def main() -> None:
          try:
              config.load_incluster_config()
          except config.ConfigException:
              config.load_kube_config()

          v1 = client.CoreV1Api()
          logger.info("Node agent started on node=%s, polling every %ds", NODE_NAME, POLL_INTERVAL)

          while True:
              ipv6 = fetch_ipv6()
              if ipv6:
                  try:
                      patch_configmap(v1, ipv6)
                  except Exception as exc:
                      logger.error("Failed to update ConfigMap: %s", exc)
              else:
                  logger.error("Could not determine IPv6 address for node %s", NODE_NAME)

              time.sleep(POLL_INTERVAL)


      if __name__ == "__main__":
          main()
      ```

2. Create a `requirements.txt`
    ```
    cloudflare>=3.1.0
    kubernetes>=29.0.0
    ```

3. Create a `Dockerfile`
    ```Dockerfile
    FROM python:3.12-slim AS builder

    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

    FROM python:3.12-slim

    LABEL org.opencontainers.image.title="cloudflare-ddns-ipv6"
    LABEL org.opencontainers.image.description="Dynamic DNS updater for IPv6 AAAA records on k3s"

    RUN apt-get update && apt-get install -y --no-install-recommends curl \
        && rm -rf /var/lib/apt/lists/*

    RUN useradd --no-create-home --shell /bin/false ddns

    WORKDIR /app
    COPY --from=builder /install /usr/local
    COPY ddns_updater.py node_agent.py ./
    RUN chmod +x ddns_updater.py node_agent.py && chown -R ddns:ddns /app

    USER ddns

    CMD ["python", "-u", "ddns_updater.py"]
    ```

4. Build and push the image to your registry:
    ```bash
    docker build -t <your-registry>/cloudflare-ddns-ipv6:latest .
    docker push <your-registry>/cloudflare-ddns-ipv6:latest
    ```

## Installation
1. Create the following directory structure for DDNS:
   ```
    ddns/
    ├── cf-secret.yml
    ├── ddns-cm.yml
    ├── ddns-node-agent.yml
    ├── ddns-rbac.yml
    ├── ddns-service-account.yml
    ├── ddns-updater.yml
    └── namespace.yml
    ```

2. Create a `cf-secret-tmp.yml` with the following content and replace the placeholders with your Cloudflare API token and zone ID:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api
      namespace: ddns-updater
    type: Opaque
    data:
      CF_API_TOKEN: <base64-encoded-api-token>
      CF_ZONE_ID: <base64-encoded-zone-id>
    ```

3. Encrypt the `cf-secret-tmp.yml` file using Sealed-Secrets and save it as `cf-secret.yml`:
    ```bash
    kubeseal --format=yaml < cf-secret-tmp.yml > cf-secret.yml && \
    rm cf-secret-tmp.yml
    ```

4. Create the `ddns-cm.yml` ConfigMap:
    ```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ddns-config
      namespace: ddns-updater
    data:
      DNS_DOMAIN: "example.com"
      NODE_NAMES: "kube-01,kube-02,kube-03,kube-04,kube-05"
      DNS_TTL: "120"
      DNS_PROXIED: "false"
      POLL_INTERVAL: "300"
      NAMESPACE: "ddns-updater"
      CONFIGMAP_NAME: "node-ipv6-addresses"
      CURL_TIMEOUT: "10"

5. Create the `ddns-node-agent.yml` DaemonSet:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: ddns-node-agent
      namespace: ddns-updater
      labels:
        app: ddns-node-agent
    spec:
      selector:
        matchLabels:
          app: ddns-node-agent
      template:
        metadata:
          labels:
            app: ddns-node-agent
        spec:
          serviceAccountName: ddns-updater
          hostNetwork: true
          dnsPolicy: ClusterFirstWithHostNet
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: node-agent
              image: <your-registry>/cloudflare-ddns-ipv6:latest
              imagePullPolicy: Always
              command:
                - "/bin/bash"
                - "-c"
                - "exec python -u node_agent.py"
              env:
                - name: NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
              envFrom:
                - configMapRef:
                    name: ddns-config
              resources:
                requests:
                  cpu: "20m"
                  memory: "32Mi"
                limits:
                  cpu: "100m"
                  memory: "64Mi"
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
    ```

6. Create the `ddns-rbac.yml` Role and RoleBinding:
    ```yaml
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: ddns-configmap-writer
      namespace: ddns-updater
    rules:
      - apiGroups: [""]
        resources: ["configmaps"]
        resourceNames: ["node-ipv6-addresses"]
        verbs: ["get", "patch", "update", "create"]

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: ddns-configmap-reader
      namespace: ddns-updater
    rules:
      - apiGroups: [""]
        resources: ["configmaps"]
        resourceNames: ["node-ipv6-addresses"]
        verbs: ["get"]

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: ddns-agent-configmap-writer
      namespace: ddns-updater
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: ddns-configmap-writer
    subjects:
      - kind: ServiceAccount
        name: ddns-updater
        namespace: ddns-updater

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: ddns-updater-configmap-reader
      namespace: ddns-updater
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: ddns-configmap-reader
    subjects:
      - kind: ServiceAccount
        name: ddns-updater
        namespace: ddns-updater
    ```

7. Create the `ddns-service-account.yml` ServiceAccount:
    ```yaml
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: ddns-updater
      namespace: ddns-updater
    ```

8. Create the `ddns-updater.yml` Deployment:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ddns-updater
      namespace: ddns-updater
      labels:
        app: ddns-updater
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ddns-updater
      template:
        metadata:
          labels:
            app: ddns-updater
        spec:
          serviceAccountName: ddns-updater
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: ddns-updater
              image: <your-registry>/cloudflare-ddns-ipv6:latest
              imagePullPolicy: Always
              envFrom:
                - configMapRef:
                    name: ddns-config
                - secretRef:
                    name: cloudflare-creds
              resources:
                requests:
                  cpu: "50m"
                  memory: "64Mi"
                limits:
                  cpu: "200m"
                  memory: "128Mi"
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
    ```

9. Create the `namespace.yml` Namespace:
    ```yaml
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: ddns-updater
    ```
10. Commit and push all the files to your Git repo and wait for flux to apply all the manifests.
