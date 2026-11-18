FluxCD GitOps pipeline for home automation<br>
**Note that this is purely for educational purposes only**<br>



### **Current Pod count: 87 (77 running, 10 completed job pods)**<br>
### **Current K8s version: v1.37.0**<br>
### **Current GPU version: 580.178.04 (NVIDIA GeForce RTX 3060)**<br>
### **Current Cilium version: v1.20.1**<br>
### **Current OS version: Ubuntu Server 22.04.5 LTS (kernel 5.15.0-202)**<br>


## **Directory Structure**<br>
**apps** - Everything Flux deploys to the cluster, one folder per area. Each folder is applied by its own Flux Kustomization (`clusters/my-cluster/apps.yaml`), so a broken manifest only stops its own folder.<br>
**clusters** - Flux bootstrap files (`flux-system`), `apps.yaml` (one Flux Kustomization per `./apps/<folder>`, all with SOPS decryption), and the separate `kubevirt` Kustomization (`kubevirt.yaml`) for `./apps/kubevirt`.<br>
**examples** - Files that are not deployed by Flux: KEDA, HAProxy/Keepalived configs, and the `research-dependency` example project.<br>

The manifests under `apps/` and `clusters/my-cluster/` are commented: every HelmRelease, Flux object, CronJob, network policy, PV and other non-trivial resource explains what it does and why it's set up that way (SOPS-encrypted files and the generated `gotk-*.yaml` are the exceptions).<br>


## **Namespaces**<br>
**kube-system** - Cluster core: Cilium, CoreDNS, cert-manager, Tailscale, the NVIDIA device plugin, Vaultwarden and the backup jobs.<br>
**cilium-spire** - SPIRE server and agent that issue workload identities for Cilium's mutual TLS.<br>
**flux-system** - The FluxCD controllers that sync this repo to the cluster.<br>
**monitoring** - Dashboards, metrics, logs and other self-hosted tools (Grafana, Prometheus, Loki, Homarr, Teslamate, etc.).<br>
**compliance** - Vulnerability and configuration scanning (Trivy) and the Headlamp dashboard.<br>
**plex** - Media server and the automation around it. Note that this Namespace should be considered a production Namespace as outside users utilize this service.<br>
**gaming** - Video game servers such as Minecraft, with Cloudflare DDNS. Note that this Namespace should be considered a production Namespace as outside users utilize this service.<br>
**dev** - Personal projects (n8n automations, local LLMs, a backtester), a KubeVirt dev VM and self-hosted GitHub Actions runners.<br>
**kubevirt** - The KubeVirt operator that runs virtual machines on the cluster.<br>
**default / kube-public / kube-node-lease** - Unused by workloads; a Cilium deny-all policy blocks all ingress and egress for any pod placed there.<br>


## **Networking overview**<br>
Cilium is the CNI and also replaces kube-proxy. Its built-in ingress controller sits behind a single LoadBalancer IP (192.168.1.254) that Cilium announces on the LAN using L2 announcements. Services are reached at `<app>.waffle-systems.com`, with HTTPS certificates issued by cert-manager from Let's Encrypt using Cloudflare DNS-01. Most apps are also exposed on a NodePort.<br>

Pod-to-pod traffic inside the media and monitoring stacks requires mutual TLS (Cilium + SPIRE), with Cilium network policies in `apps/network/cilium-network-mTLS.yaml` defining what may talk to what. Remote access to the LAN, service and pod networks is through a Tailscale subnet router.<br>


# **In-depth overview of the pods on each Namespace**<br>

## **kube-system**<br>

cilium / cilium-envoy / cilium-operator / hubble-relay / hubble-ui - Cilium is the CNI of choice. It is not managed by FluxCD; it is installed with Helm (see the setup steps below) with kube-proxy replacement, the ingress controller, Gateway API, L2 announcements, SPIRE mutual authentication, Prometheus metrics and Hubble enabled. Hubble UI is reachable on NodePort 30007 and at hubble.waffle-systems.com.<br>

cert-manager - deployed via the jetstack cert-manager chart. Uses the `letsencrypt-prod` ClusterIssuer with a Cloudflare DNS-01 solver to automatically request and renew TLS certificates for the Ingresses in `apps/network/cert-manager.yaml`.<br>

tailscale-subnet-router - advertises the home LAN (192.168.1.0/24), the service CIDR and the pod CIDR to the tailnet, so the cluster and home network can be reached remotely without exposing them to the internet.<br>

nvidia-device-plugin - exposes the GPU to Kubernetes as the `nvidia.com/gpu` resource. Time slicing splits the single GPU into 4 schedulable replicas so that Plex/Tdarr, Ollama and other workloads can share it. A `nvidia` RuntimeClass is also defined.<br>

etcd-backup - a CronJob that runs every Sunday at 03:00, takes an etcd snapshot and saves it to an HDD at `/mnt/backup/etcd`, so the cluster can be recovered if it becomes corrupt.<br>

weekly-server-backup - a CronJob that runs every Sunday at 02:00 and rsyncs `/home/server` into a dated folder on the backup HDD (`/mnt/backup`), deleting copies older than 30 days.<br>

vaultwarden - a self-hosted Bitwarden-compatible password manager at vault.waffle-systems.com, with signups disabled.<br>

coredns / etcd / kube-apiserver / kube-controller-manager / kube-scheduler / kube-proxy / metrics-server - standard Kubernetes control-plane and cluster add-on pods, not managed by FluxCD.<br>


## **cilium-spire**<br>

spire-server / spire-agent - installed by Cilium. They issue SPIFFE identities to workloads, which Cilium uses to enforce mutual TLS between pods. The server stores its data on a hostPath PersistentVolume.<br>


## **flux-system**<br>

source / kustomize / helm / notification / image-reflector / image-automation controllers - the FluxCD controllers. They pull this repo, apply the per-folder Kustomizations (decrypting SOPS secrets with the `sops-age` key), install HelmReleases, and scan container registries to automatically bump image tags marked with `$imagepolicy` comments.<br>


## **monitoring**<br>

homarr - deployed via a deployment.yaml file, this pod is the backbone of the home network. Homarr allows for links and integrations to be placed on a neat and tidy web interface for ease of use when running multiple pods. Image updates are automated by Flux.<br>

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/65d5cd66-4762-4385-b307-4ed8fcc9ac70" /><br>

prometheus / grafana / alertmanager / kube-state-metrics / node-exporter - deployed via the kube-prometheus-stack chart from prometheus-community on the helmrelease.yaml. Prometheus collects metrics and Grafana displays them. Alerts can be integrated using Alertmanager, allowing for emails, webhooks, etc. when certain thresholds are met. Grafana's admin credentials are a secret encrypted with SOPS/AGE and stored in this FluxCD project; it is decrypted in the cluster by a private key stored independently on the K8s cluster. Grafana's dashboards, datasources and alerting are provisioned from a hostPath folder. Prometheus scrapes `/metrics` endpoints through ServiceMonitors from applications such as Trivy and DCGM Exporter.<br>

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/3d6d9223-cb5b-40dc-9cf7-f02719c08313" />


loki / alloy - Loki is deployed via the loki chart from grafana-community in single-binary mode, along with its gateway, canary and memcached chunk/results caches. Grafana Alloy runs as a DaemonSet that reads the host's `/var/log/syslog` and pushes it to Loki. Loki is then used as a datasource in Grafana so logs can be searched and shown on dashboards. Loki stores its data on an emptyDir, so logs do not survive a pod restart.<br>

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/83bf7066-085d-483e-8119-805d4d1cf087" />


dcgm-exporter - deployed via the dcgm-exporter chart from NVIDIA on the helmrelease.yaml. This pod exposes GPU metrics, and through a ServiceMonitor Prometheus ingests the data.<br>

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/d752749f-092a-4f83-b499-04dd9c36abbb" />


prometheus-adapter - exposes Prometheus container and node CPU/memory data through the Kubernetes resource metrics API.<br>

teslamate / teslamate-postgresql / mosquitto - Teslamate is a self-hosted data logger for Tesla vehicles, deployed via its Helm chart with a bundled PostgreSQL database on a hostPath PersistentVolume. Mosquitto is the MQTT broker Teslamate publishes vehicle data to.<br>

glances - a system monitoring web UI showing host CPU, memory, disk, network and process stats (runs with the host's PID namespace and a read-only mount of `/`).<br>

snapotter - SnapOtter, a self-hosted image and file processing toolkit (with its own embedded PostgreSQL and Redis), storing its data on a hostPath folder.<br>

mealie - a self-hosted recipe manager at mealie.waffle-systems.com (SQLite on a hostPath folder). Its "Import with AI" feature uses the cluster's Ollama. Image updates are automated by Flux.<br>


## **compliance**<br>

trivy-operator / trivy-server - deployed via the trivy-operator and trivy charts from Aqua Security. The operator scans workloads in the cluster for vulnerabilities, initially and whenever new pods arrive, and runs in ClientServer mode against a central trivy-server that holds the vulnerability database. A ServiceMonitor lets Prometheus collect its data so the results can be viewed in Grafana.<br>

<img width="1340" height="486" alt="image" src="https://github.com/user-attachments/assets/d576d94e-a4c9-45e9-a77c-9ed7e991d102" />

headlamp - the Headlamp Kubernetes dashboard at headlamp.waffle-systems.com, with the Trivy plugin for browsing vulnerability, config-audit and compliance (CIS / NSA / PSS) reports. It logs in with a read-only service account token (no cluster-admin, no access to Secrets).<br>

kubescape - disabled since 2026-11-14 (Trivy covers the same checks). Its config is kept as comments in `apps/compliance` so it can be turned back on.<br>


## **plex**<br>

plex - the media server that ties all of this in, allowing users to watch movies or shows via the Plex app. Media is read from the RAID at `/mnt/Media`.<br>

sonarr - an automation tool for managing TV show downloads, integrating with download clients to grab episodes as soon as they're available.<br>

radarr - similar to Sonarr but designed for movies, automating the process of finding, downloading, and organizing films.<br>

prowlarr / flaresolverr - Prowlarr is an indexer manager that syncs torrent trackers to Sonarr and Radarr. FlareSolverr runs as a sidecar in the same pod to get past Cloudflare challenges on some indexers.<br>

qbittorrent / gluetun - qBittorrent is the download client used by Sonarr and Radarr. All of its traffic goes through a Gluetun WireGuard VPN sidecar, and an init container blocks qBittorrent from starting until the VPN is confirmed working.<br>

qbittorrent-restarter - a CronJob that restarts the qBittorrent deployment every Sunday at 00:00, using the official `registry.k8s.io/kubectl` image (kept on the cluster's minor version by Flux image automation).<br>

tdarr-server / tdarr-node - Tdarr is a media transcoding tool that automatically converts video files to optimized formats for Plex. The node uses the GPU for transcoding and waits for the server to be up before starting.<br>

seerr - a web-based request management system for Plex, allowing users to request new movies or TV shows, which are then sent to Sonarr and Radarr.<br>

cloudflared - runs a Cloudflare Tunnel, giving outside users access to selected services without opening router ports.<br>


## **gaming**<br>

minecraft - a NeoForge 1.21.8 Minecraft server configured through environment variables. The pod is exposed via a NodePort, and the router port forwards outside traffic on the Minecraft port to the server.<br>

cloudflare-ddns - checks whether the network's public IP address has changed and, if so, updates the `minecraft.waffle-systems.com` record in Cloudflare.<br>

project-zomboid - a Project Zomboid server manifest is kept in the repo but is currently disabled in the gaming kustomization.<br>


## **dev**<br>

n8n / task runner / ollama - n8n workflow automation (job search, shop tools, blog drafts, Discord posts) at n8n.waffle-systems.com, with its JavaScript task runner and Ollama in the same pod. Ollama uses the GPU to serve local LLMs. Flux follows n8n's `stable` tag by digest and Ollama's release tags.<br>

n8n-sandbox-api / n8n-sandbox-runner - n8n's code sandbox service and its Docker-in-Docker runner, connected with mutual TLS.<br>

searxng - a private metasearch engine used by the n8n workflows for web searches (cluster-internal only).<br>

mongodb - the database for n8n's AI memory and the workflows' saved data.<br>

discord-assistant - a small relay that forwards Discord messages to an n8n workflow and posts the reply.<br>

backtest-api / backtest-report-web / data-update - a trading-strategy backtester (`apps/dev/backtest`, image from GHCR) with a web UI at backtest.waffle-systems.com, a static report site and a weekday CronJob that downloads market data.<br>

devbox (virt-launcher) - a KubeVirt VirtualMachine running Fedora 45 (containerDisk, so its disk resets on restart) with 2 cores, 2Gi of memory and EFI Secure Boot, configured by a cloud-init secret.<br>

arc-gha-rs-controller / my-runner-set-listener - Actions Runner Controller. It runs self-hosted GitHub Actions runners (Docker-in-Docker mode) for the Finance-Friend repository, scaling runner pods up when jobs are queued.<br>


## **kubevirt**<br>

virt-operator / virt-api / virt-controller / virt-handler - KubeVirt v1.9.1, installed from the upstream release manifests through its own Flux Kustomization (`clusters/my-cluster/kubevirt.yaml`, path `./apps/kubevirt`); `apps.yaml` does not include that folder, so the operator and its CRDs have exactly one owner. KubeVirt only supports upgrading one minor version at a time, from the latest patch of each minor (e.g. v1.8.4 → v1.9.1), so bump the version in `apps/kubevirt/kustomization.yaml` one step at a time. It lets the cluster run virtual machines (such as the devbox) alongside containers, which is useful for testing security implementations.<br>





# **Initial setup of the Cluster**<br>

Step 1 - Install Cilium with the features this cluster uses:<br>
```bash
cilium install --version 1.20.1 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.1.200 \
  --set k8sServicePort=6443 \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set operator.replicas=1 \
  --set ingressController.enabled=true \
  --set ingressController.loadbalancerMode=shared \
  --set ingressController.service.type=LoadBalancer \
  --set l2announcements.enabled=true \
  --set gatewayAPI.enabled=true \
  --set prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set authentication.enabled=true \
  --set authentication.mutual.spire.enabled=true \
  --set authentication.mutual.spire.install.enabled=true
```

Step 2 - Create the age key and the sops-age secret:<br>
```bash
sudo apt install sops age
mkdir -p ~/.config/sops/age
age-keygen -o age.agekey
```

  Make a yaml for the sops key. This tells sops which key to use to encrypt, so you don't have to use flags every time. You can push this to GitHub as it's just a public key (see `examples/.sops.yaml`):<br>
```yaml
# .sops.yaml
creation_rules:
  - encrypted_regex: '^(data|stringData)$'
    age: YourPublicKeyGoesHere
```

  Add the private key to the cluster, so the files can be decrypted once they get into the cluster:<br>
```bash
# Ensure you are in the folder containing age.agekey
kubectl create namespace flux-system

cat age.agekey | kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

  Tell FluxCD to use the key. Every Kustomization in `clusters/my-cluster/apps.yaml` (one per folder in `./apps`) has the `decryption` block, e.g. plex:<br>
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: plex
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/plex
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Step 3 - Install the Flux CLI and bootstrap FluxCD onto the cluster:<br>

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version

flux bootstrap github \
  --owner=<your-github-username> \
  --repository=FluxCD \
  --branch=main \
  --path=clusters/my-cluster \
  --components-extra=image-reflector-controller,image-automation-controller \
  --read-write-key

flux get sources git -n flux-system
flux get kustomizations -A
```

  Always pass both `--components-extra` and `--read-write-key` when re-running bootstrap: without the extra components Flux removes the image automation controllers and their CRDs, and without `--read-write-key` the new deploy key is read-only, so image automation can no longer push.<br>
