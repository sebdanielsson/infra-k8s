# Setup Talos configuration for Proxmox cluster

## Prerequisites

Install Talos CLI tool:

```sh
brew install siderolabs/tap/talosctl
```

Install kubectl:

```sh
brew install kubectl
```

Install Flux CLI:

```sh
brew install fluxcd/tap/flux
```

Install Helm:

```sh
brew install helm
```

Download Talos installer ISO to Proxmox server:

- Go to [Image Factory](https://factory.talos.dev/) and generate an ISO image for Talos.
- Upload the ISO image to Proxmox using the URL provided by the Image Factory.

## Create VMs in Proxmox

Do this step via the Proxmox web interface.

Create 3 VMs for the cluster with the following specs:

- Name: talos-0, Machine: Q35, QEMU Agent: Check, CPU: 4, RAM: 4096MB, Disk: 32GB, Network: vmbr0, ISO Image: Talos installer ISO
- Name: talos-1, Machine: Q35, QEMU Agent: Check, CPU: 4, RAM: 4096MB, Disk: 32GB, Network: vmbr0, ISO Image: Talos installer ISO
- Name: talos-2, Machine: Q35, QEMU Agent: Check, CPU: 4, RAM: 4096MB, Disk: 32GB, Network: vmbr0, ISO Image: Talos installer ISO

## Setup HA Control Plane

Boot the 3 VMs and set their IP addresses accordingly. Update the IPs below if necessary.

```sh
export CP_NODE_1_IP=192.168.1.231
export CP_NODE_2_IP=192.168.1.240
export CP_NODE_3_IP=192.168.1.182
```

### Control Plane Node 1 - talos-0

```sh
talosctl gen config talos-proxmox-cluster https://$CP_NODE_1_IP:6443 --output-dir _out --install-image factory.talos.dev/installer/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515:v1.11.5
talosctl apply-config --insecure --nodes $CP_NODE_1_IP --file _out/controlplane.yaml
```

### Create Worker Node - talos-1

```sh
talosctl apply-config --insecure --nodes $CP_NODE_2_IP --file _out/worker.yaml
```

### Create Worker Node - talos-2

```sh
talosctl apply-config --insecure --nodes $CP_NODE_3_IP --file _out/worker.yaml
```

### Bootstrap etcd on Control Plane Node 1

```sh
export TALOSCONFIG="_out/talosconfig"
talosctl config endpoint $CP_NODE_1_IP
talosctl config node $CP_NODE_1_IP
talosctl bootstrap
```

### Install Metallb on the Cluster

```sh
curl https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml -o metallb.yaml
kubectl apply -f metallb.yaml
```

### Bootstrap FluxCD on the Cluster

```sh
flux bootstrap github \
  --token-auth \
  --owner=sebdanielsson \
  --repository=infra-k8s \
  --branch=main \
  --path=clusters/talos \
  --personal
```
