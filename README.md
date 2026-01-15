FluxCD GitOps pipeline for home automation<br>
**Note that this is purely for educational purposes only**<br>



### **Current Pod count: 68**<br>
### **Current K8s version: v1.34.0**<br>
### **Current GPU version: 535.274.02 (NVIDIA)**<br>
### **Current OS version: Ubuntu Server 5.15.0-164**<br>


## **Directory Structure**<br>
**apps** - Where all current implementations of Fluxs pipeline exists on the cluster.<br>
**clusters** - Administrative files not pertaining to the actual applications.<br>
**examples** - Example files not currently implemented in via FluxCD.<br>
**research-dependency** - Another example project not currently implemented.<br>


## **Namespaces**<br>
**infrastructure** - for primarily dashboards, logs, and metric scrapers.<br>
**infrastructure-network** - for ingress traffic and certs for HTTPS.<br>
**kubevirt** - for virtual environments where you can test security implementations.<br>
**gaming** - for hosting video game servers such as Minecraft, on a DDNS (DuckDNS). Note that this Namespace should be considered a production Namespace as outside users utilize this service.<br>
**plex** - for hosting media content. Note that this Namespace should be considered a production Namespace as outside users utilize this service.<br>
**keda** - for scaling services based on utlization, such as CPU utilization scaling.<br>
**trivy-system** - for pod based vulnerability scanner. These scanning pods are attached to Grafana in order to view the scans more collectively.<br>


# **In-depth overview of the pods on each Namespace**<br>

## **kube-system**<br>

cilium - Cilium is the CNI of choice, although not implemented into the FluxCD project, cilium is launched with HubbleUI, and some other enabled features with the command:<br>
```bash <br>
cilium upgrade --namespace kube—system   --set ingressController.enabled=true   --set ingressController.service.type=LoadBalancer   --set ingressController.loadBalancerMode=dedicated   --set prometheus.enabled=true   --set hubble.enabled=true   --set hubble.relay.enabled=true   --set hubble.relay.rollOutPolicy=Always   --set hubble.ui.enabled=true   --set hubble.ui.rollOutPolicy=Always   --set hubble.ui.backend.clusterName=default   --set hubble.ui.backend.address=relay.kubesystem.svc:4245   --set gatewayAPI.enabled=true
```


etcd-backup - a cron laucnhed pod that conducts a backup of etcd and puts it in a HDD so that in the event the cluster is corrupt, we can still recover from it.

nvidia - Allows Kubernetes to run workloads that require GPU acceleration by leveraging the NVIDIA device plugin for Kubernetes, which exposes GPUs as resources that can be scheduled by the kubelet. GPUs are utilized by specifying GPU resource requests in pod specifications, enabling workloads like machine learning or data processing to offload compute-intensive tasks to the GPU.


## **infrastructure**<br>

homarr - deployed via a deployment.yaml file, this pod is the backbone of the home network. homarr allows for links, and integrations to be placed on a neat and tidy web interface for ease of use when running multiple pods <br>

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/65d5cd66-4762-4385-b307-4ed8fcc9ac70" /><br>

prometheus / grafana / alertmanager - deployed via the kube-prometheus-stack chart from creator prometheus-community on the helmrelease.yaml, these pods aim to collect metrics via prometheus and display them with grafana. Alerts can be integrated using the alertmanager section, allowing for emails, webhooks, etc. where servers can be monitored if certain thresholds are met. Grafana is setup to where a secret is encrypted with SOPS/AGE and stored on this FluxCD project. The encryption is then decrypted via a private key stored independently on the K8s cluster. Prometheus is a metrics collector and scraper, that with a service called serviceMonitor it can collect metrics universally from /metrics on applications such as Trivy, Teslamate, DCGM Exporter, etc.

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/3d6d9223-cb5b-40dc-9cf7-f02719c08313" />


loki / promtail - deployed via the loki chart from creator grafana-charts on the helmrelease.yaml, these pods aim to scrape logs, and throw them in a backend database to be displayed on grafana. Unlike prometheus that collects logs for applications, loki scrapes all the logs on servers, and throws it in a backend database known as promtail. Promtail then is connected to grafana via a datasource and can be fully integrated and toyed with on the dashboards that utilize the promtail datasource.

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/83bf7066-085d-483e-8119-805d4d1cf087" />


dcgm-exporter - deployed via the dcgm-exporter chart from creator dcgm-exporter on the helmrelease.yaml, this pod is utilized to scrape GPU metrics and due to a serviceMonitor....prometheus ingests the data. 

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/d752749f-092a-4f83-b499-04dd9c36abbb" />



## **infrastructure-network**<br>

metallb - These pods give your baremetal cluster the ability to create LoadBalancer services. For this configuration the assigned IP is 192.168.1.254, and is reachable IP addresses to your services so traffic can actually enter your cluster from the outside world.

ingress-nginx - This pod acts as a reverse proxy and traffic cop for HTTP/HTTPS. It takes the traffic allowed in by metallb and routes it to specific internal services based on the specified domain name.

cert-manager - These pods automate security by talking to authorities like Let's Encrypt. They automatically request, validate, and renew SSL/TLS certificates so your Ingress endpoints stay secure (HTTPS) without manual work.


## **kubevirt**<br>




## **gaming**<br>

minecraft - A simple pod that hosts minecraft based on environment variables. The pod is exposed via a nodeport, and the router port forwards outside traffic to the server on the specific Minecraft port to the server.

duck-dns-cron - A pod that utilizes a secret token and domain to see if the public IP address of my network has changed. If so, DuckDNS gets updated with the new IP. This cron runs every 5 minutes.


## **plex**<br>

sonarr - is an automation tool for managing TV show downloads, integrating with clients to grab episodes as soon as they're available.

radarr - is similar to Sonarr but designed for movies, automating the process of finding, downloading, and organizing films.

qbittorrent - is an open-source client that offers a clean interface and support for automation tools like Sonarr and Radarr.

qbitmanage - a management pod that can handle the qbittorrent download issues. 

jackett - is a proxy server that enables clients like qBittorrent to interface with various public and private trackers.

tdarr - is a media transcoding tool for automatically converting video files to optimized formats, often used in conjunction with Plex.

overseerr - is a web-based request management system for Plex or Jellyfin, allowing users to request new movies or TV shows for download.

plex - ties all of this in to allow users to watch movies or shows for free via the plex app.


## **keda**<br>

keda - Enables Kubernetes workloads to scale based on external events, such as messages in a queue or metrics from a custom source. It extends the native Horizontal Pod Autoscaler (HPA) by integrating event-driven triggers, allowing for dynamic scaling based on demand in real-time, while supporting a wide range of event sources like Azure, AWS, or custom metrics.

## **trivy-system**<br>

trivy - deployed via the trivy-operator chart from creator aquasec/trivy on the helmrelease. These pods scan the entire cluster initially and when new pods arrive. Trivy is set to automatically update when a new release is issued and deploys a service monitor so that prometheus can collect on its data. 

<img width="1340" height="486" alt="image" src="https://github.com/user-attachments/assets/d576d94e-a4c9-45e9-a77c-9ed7e991d102" />





# **Initial setup of the Cluster**<br>

Step 1 - Get Cilium up with the previously utilized command:<br>
```bash <br>
cilium upgrade --namespace kube—system   --set ingressController.enabled=true   --set ingressController.service.type=LoadBalancer   --set ingressController.loadBalancerMode=dedicated   --set prometheus.enabled=true   --set hubble.enabled=true   --set hubble.relay.enabled=true   --set hubble.relay.rollOutPolicy=Always   --set hubble.ui.enabled=true   --set hubble.ui.rollOutPolicy=Always   --set hubble.ui.backend.clusterName=default   --set hubble.ui.backend.address=relay.kubesystem.svc:4245   --set gatewayAPI.enabled=true
```

Step 2 - install the secret sops-age in kube-system:<br>
```bash <br>
sudo apt install sops age
mkdir -p ~/.config/sops/age
age-keygen -o age.agekey
```

  Make a yaml for the sops key, this tells sops what key to use to encrypt, so you dont have to use flags everytime. You can push this to Github as its just a public key<br>
```bash <br>
# .sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: YourPublicKeyGoesHere
```

  Add the private key to the cluster, so you can decrypt the files once they get into the cluster<br>
```bash <br>
# Ensure you are in the folder containing age.agekey
kubectl create namespace flux-system

cat age.agekey | kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

  Tell FluxCD to use the key<br>
```bash <br>
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/my-cluster
  sourceRef:
    kind: GitRepository
    name: flux-system
  # Add this section:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Step 3 - install FluxCD onto the cluster:<br>

```bash <br>
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
helm repo add fluxcd https://charts.fluxcd.io
helm repo update
helm install helm-operator fluxcd/helm-operator --namespace flux-system

flux bootstrap github \
--owner=<your-github-username> \
--repository=FluxCD \
--branch=main \
--path=clusters/my-cluster \ 
--components=source-controller,kustomize-controller,helm-controller,notification-controller,image-reflector-controller,image-automation-controller \ 
--watch-all-namespaces 

kubectl get gitrepositories -n flux-system
flux get syncs --namespace flux-system
helm install flux-helm-operator fluxcd/helm-operator --namespace flux-system
kubectl logs -n flux-system deployment/flux
```
