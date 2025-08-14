# What is k3s
k3s is a `lightweight, CNCF-certifed kubernetes distrubtion` (by SUSE/Rancer). It is a simple small binary that is tuned for edge/IoT/labs. Runs as a real multi-node 8s across multiple Linux machines/vMs.

## k3s vs miniube

* What is minikube: minikube is a `local kubernetes cluster` that spins up a cluster on your single computer(MacOS/Windows/Linux) using a VM or contaier driver, that is mainly for development or testing.

* Compare the two

Aspect | k3s | minikube
--- | --- | ---
primary use|Edge/small prod, homelab|local dev & demo
nodes|True multi-node across hosts|Signle host; multi-node `simulated` on single host
footprint|Very small (single binary)|Small, but adds VM/driver overhead
Runtime|containerd (built-in)|Dokcer/containerd/VM (driver-dependent)
CNI/Ingress|Flanel + Traefik by default (both optional)|Addons (e.g. nginx-ingress)
LoadBalancer|Built-in `klipper-lb`|__miniube tunnel__ orMetalLB addon
Storage|Local Path Provisioner bundled|HostPath storage addon
HA/Datastore|SQLite by default; embedded/external etcd for HA|Not aimed at HA/prod
OS|Linux (native). Use VM/WSL2 on Mac/Windows|Run on Mac/Windows/Linux(cluster inside VM/container)

## k3s & istio
k3s is a CNCF-certified Kubernetes, so `istio works on k3s/k3d`. Typical setups:

### standard (sidecar) intall

```bash
    istioctl x precheck
    istioctl install -y --set profile=default
    kubectl label ns default istio-injection=enabled
```

### With istio CNI (recommended on k3s)

This can avoide init-container iptables tweaks and plays nice with k3s

```bash
   istioctl install -y --set profile=default --set components.cni.enabled=true
```

### Ingress choice on k3s

* Use LB: __--set values.gateways.istio-ingressgateway.type=LoadBalancer__ (use k3s `klipper-lb)
* Or NodePort if you prefer
* Optional: install k3s with __--disable traefik__ to avoide port 80/443 conflicts

### CNI

* Works with default `flannel` or with `cilium` (if you replace flannel)

### Ambient (sidecarless)

* Supported in recent istio; install the ambient profile if want ztunnel/waypoints

## Work with Azure pipeline

One can defined a `self-hosted Azuer Pipelines agent` that can reach your local k3s API, but don expose k3s on the public internet

### Give CI a kubeconfig (namespace-scoped)

```bash
kubectl create ns dev
kubectl -n dev create sa ci
kubectl create clusterrolebinding ci-admin --clusterrole=cluster-admin --serviceaccount=dev:ci
# make a short-lived token + kubeconfig for the agent
kubectl -n dev create token ci > ci.token
kubectl config set-credentials ci --token="$(cat ci.token)"
kubectl config set-context ci --cluster=$(kubectl config current-context) --user=ci --namespace=dev
kubectl config use-context ci
kubectl config view --minify --flatten > kubeconfig-ci
```

### Minimal recipe

```yaml
pool: { name: SelfHosted }  # your agent pool
stages:
- stage: Build
  jobs:
  - job: BuildPush
    steps:
    - script: |
        docker build -t $(ACR_NAME).azurecr.io/app:$(Build.BuildId) .
        az acr login -n $(ACR_NAME)
        docker push $(ACR_NAME).azurecr.io/app:$(Build.BuildId)

- stage: Deploy
  jobs:
  - job: k3s
    steps:
    - task: DownloadSecureFile@1
      inputs: { secureFile: 'kubeconfig-k3s' }
    - task: KubectlInstaller@0
      inputs: { kubectlVersion: 'latest' }
    - script: |
        export KUBECONFIG=$(Agent.TempDirectory)/kubeconfig-k3s
        kubectl -n demo apply -f k8s/      # or: helm upgrade --install ...
```


## Configure a k3s multi-nodes

* 1 server + 3 agents (4 nodes)
* Light workloads can strech to 1 + 4; for istio/monitoring keep this to 1 + 2/1 + 3
* Example:

  ```bash
  k3d cluster create m2lab \
    -s 1 -a 3 \
    -p "80:80@loadbalancer" -p "443:443@loadbalancer" \
    --k3s-arg "--disable=traefik@server:*"
  ```
  Note: for Linux/WSL2, replace k3d with k3s

* Use istio with CNI

  ```bash
  istioctl install -y \
    --set profile=default \
    --set components.cni.enabled=true \
    --set values.gateways.istio-ingressgateway.type=LoadBalancer
  ```

