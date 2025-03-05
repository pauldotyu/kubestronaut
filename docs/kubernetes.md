## Overview

This workshop intentionally uses Kubernetes v1.31.6 to provide cluster upgrade practice opportunities. While current exams are based on Kubernetes v1.32, the core concepts remain consistent between versions.

As part of the cluster setup, you'll install containerd for the container runtime, [gVisor](https://gvisor.dev/) for container isolation, Cilium as the CNI plugin, MetalLB for mimicking cloud load balancers, and Ingress-Nginx Controller - all well-documented, popular choices that you can substitute if preferred.

## Prepare the control and worker nodes

For each of the nodes (control and workers), you'll need to install the necessary packages and configure the system. Each node will need to have the following installed: 

- containerd
- gVisor
- crictl
- kubeadm
- kubelet
- kubectl

To begin, SSH into the control node and switch to the root user.

```bash
sudo -i
```

!!! note
    Most of the tasks you perform will be as the root user. This is avoid having to use `sudo` for every command.

### Disable swap

To avoid Kubernetes data such as contents of Secret object being written to tmpfs, "swap to disk" must be disabled. 

!!! info
    See [Tim Hokin's comment](https://discuss.kubernetes.io/t/swap-off-why-is-it-necessary/6879) and the [kubeadm prerequisites](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) to better understand why swap must be disabled.

**Reference:** [https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory](https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory)

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### System updates

Update the system and install necessary packages.

```bash
apt-get update && apt-get upgrade -y
```

### Install gVisor

gVisor is is used to provide an additional layer of security for containers. It uses runsc which is a lightweight, portable, and secure container runtime that implements the gVisor runtime interface.

Install the dependencies for runsc.

```bash
apt-get install -y \
apt-transport-https \
ca-certificates \
curl \
gnupg
```

Add the gVisor repository and install runsc.

```bash
curl -fsSL https://gvisor.dev/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | sudo tee /etc/apt/sources.list.d/gvisor.list > /dev/null
sudo apt-get update && sudo apt-get install -y runsc
```

!!! note
    runsc will be configured as the runtime for containerd in the next steps.

**Reference:** [https://gvisor.dev/docs/user_guide/containerd/quick_start/](https://gvisor.dev/docs/user_guide/containerd/quick_start/)

### Install containerd

Kubernetes uses the Container Runtime Interface (CRI) to interact with container runtimes and containerd is the container runtime that it uses (it was dockerd in the past).

Install containerd on each node from the Docker repository and configure it to use systemd as the cgroup driver.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)

Add docker gpg key and repository.

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update packages and install containerd.

```bash
apt-get update
apt-get install containerd.io -y
```

Configure containerd to use systemd as the cgroup driver, use systemd cgroups, and use runsc as the runtime.

```bash
# generate default config
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
# update to use systemd cgroup driver
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
# configure containerd to use runsc
sed -e 's/shim_debug = false/shim_debug = true/g' -i /etc/containerd/config.toml
sed -e '/\[plugins."io.containerd.grpc.v1.cri".containerd.runtimes\]/a \\n        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]\n          runtime_type = "io.containerd.runsc.v1"' /etc/containerd/config.toml
# restart containerd
systemctl restart containerd
systemctl enable containerd
```

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

Update containerd to load the overlay and br_netfilter modules. This is necessary for the overlay network to work.

```bash
cat << EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Update kernel network settings to allow traffic to be forwarded.

```bash
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional)

Load the kernel modules and apply the sysctl settings to ensure the changes are used.

```bash
modprobe overlay
modprobe br_netfilter
sysctl --system
```

Verify containerd is running.

```bash
systemctl status containerd
```

### Install kubeadm, kubelet, and kubectl

The following tools are necessary for a successful Kubernetes installation:

- **kubeadm**: the command to bootstrap the cluster.
- **kubelet**: the component that runs on all of the machines in the cluster and does things like starting pods and containers.
- **kubectl**: the command line tool to interact with the cluster.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

Add the Kubernetes repository.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

Update the system, install the Kubernetes packages, and hold packages so they don't get unintentionally updated.

```bash
apt-get update
apt-get install -y kubelet=1.31.6-1.1 kubeadm=1.31.6-1.1 kubectl=1.31.6-1.1
apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet.

```bash
systemctl enable --now kubelet
```

### Install critctl

CRI-O is an implementation of the Container Runtime Interface (CRI) used by the kubelet to interact with container runtimes. Install crictl by downloading the binary to the system.

```bash
export CRICTL_VERSION="v1.31.1"
export CRICTL_ARCH=$(dpkg --print-architecture)
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz
tar zxvf crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz -C /usr/local/bin
rm -f crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz
```

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o)

Verify crictl is installed.

```bash
crictl version
```

### Configure crictl

Configure crictl to use the containerd socket by default to avoid seeing a warning each time you run crictl commands.

```sh
crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock
```

**Reference:** [https://github.com/kubernetes-sigs/cri-tools/issues/868#issuecomment-1926494368](https://github.com/kubernetes-sigs/cri-tools/issues/868#issuecomment-1926494368)

!!! danger
    Before you move on, jump back to the [Prepare the control and worker nodes](#prepare-the-control-and-worker-nodes) section and repeat these steps for each worker node.

## Kubernetes control plane

The control node is where the Kubernetes control plane components will be installed. This includes the [API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/), [controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/), [scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/), and [etcd](https://etcd.io/). The control node will also run the [CNI plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) to provide networking for the cluster.

### Install with kubeadm

Using kubeadm, install the Kubernetes with the `kubeadm init` command. This will install the control plane components and create the necessary configuration files.

Make sure you are back in the control node as the root user and run the following command to initialize the cluster.

```bash
kubeadm init --kubernetes-version 1.31.6 --pod-network-cidr 10.21.0.0/16 --v=5
```

!!! note
    This can take a few minutes to complete. Be patient and wait for the process to finish.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

Export the kubeconfig file so the root user can access the cluster.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Install Cilium CNI plugin

Kubernetes requires a Container Network Interface (CNI) plugin to provide networking for the cluster. Cilium will be used as the CNI plugin for the cluster. Cilium is a powerful, efficient, and secure CNI plugin that provides network security, observability, and troubleshooting capabilities.

**Reference:** [https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

Install the Cilium CLI.

!!! note
    The Cilium CLI version will be pinned to `v0.16.24`. To get the latest version, visit the [Cilium CLI releases page](https://github.com/cilium/cilium-cli/releases) and update the version number.


Set the Cilium version and architecture.

```bash
export CILIUM_VERSION="v0.16.24"
export CILIUM_ARCH=$(dpkg --print-architecture)
```

Download the Cilium CLI binary its sha256sum and verify sha256sum to ensure the binary isn't corrupted.

```bash
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_VERSION}/cilium-linux-${CILIUM_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CILIUM_ARCH}.tar.gz.sha256sum
```

Move binary to proper location and remove tarball file.

```bash
tar xzvf cilium-linux-${CILIUM_ARCH}.tar.gz -C /usr/local/bin
rm cilium-linux-${CILIUM_ARCH}.tar.gz{,.sha256sum}
```

Verify the Cilium CLI is installed.

```bash
cilium version --client
```

Install Cilium CNI plugin.

```bash
cilium install --version 1.17.1
```

Run the following command to monitor the Cilium installation progress.

```bash
cilium status --wait
```

!!! note
    This process can take a few minutes and you'll initially see errors and warnings in the output. They will eventually resolve so be patient.

Within a few minutes, you should see output like this which shows the Cilium CNI plugin is installed and running.

```text
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium             Running: 1
                       cilium-envoy       Running: 1
                       cilium-operator    Running: 1
Cluster Pods:          0/2 managed by Cilium
Helm chart version:
Image versions         cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
```

### Cluster verification

Exit the root shell and run the following commands to configure kubectl for your normal user account.

```bash
exit
```

Configure kubectl for your normal user account.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify you can connect to the cluster.

```bash
kubectl get nodes
```

You should see the control node listed as **Ready**. This means the CNI plugin is running and the control node is ready to accept workloads.

Print the join command and run it on the worker nodes.

```bash
kubeadm token create --print-join-command
```

!!! danger
    Copy the join command to your clipboard or a text file as you'll need it for the next step.

## Join the worker nodes

Log into each worker node, make sure you are in the root shell, and paste and run the join command that you copied from the control node.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)

If your shell doesn't show `root@worker`, run `sudo -i` to switch to the root user.

Run a command similar to the following on each worker node.

```bash
kubeadm join 172.16.25.132:6443 --token xxxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxx
```

After the worker nodes have joined the cluster, log back into the control node and verify the nodes are listed as **Ready**.

```bash
kubectl get nodes -w
```

Wait until all the worker nodes are listed as **Ready** then press `Ctrl+C` to exit the watch.

!!! note
    At this point you can log out of the worker nodes and probably don't need to log into them again. You can work with cluster from the control node.

## Install and configure tools

Log back into the control node as your normal user to install some handy tools and configure kubectl. 

### Configure kubectl completion and alias

These configurations are optional but will make your life easier when working with Kubernetes.

Set kubectl alias so you can use `k` instead of `kubectl` and add kubectl completion.

```bash
cat << EOF | tee -a ~/.bashrc
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
EOF
```

**Reference:** [https://kubernetes.io/docs/reference/kubectl/quick-reference/](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

Reload the bash profile.

```bash
source ~/.bashrc
```

### Configure vim

Vim will be the main editor you work with throughout the exams, it is best to configure .vimrc to help with editing YAML files.

```bash
cat << EOF | tee -a ~/.vimrc
set tabstop=2
set expandtab
set shiftwidth=2
EOF
```

### Install Helm

[Helm](https://helm.sh) is a package manager for Kubernetes that helps you manage Kubernetes applications. Helm uses a packaging format called charts which are a collection of files that describe a related set of Kubernetes resources.

Run the following commands to install Helm.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm ./get_helm.sh
```

**Reference:** [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

### Install etcdctl

etcd is the critical distributed key-value store that serves as Kubernetes' primary database, storing all cluster configuration, state, and metadata.

![Diagram showing etcd's central role in Kubernetes architecture](https://user-images.githubusercontent.com/5873433/234062738-14844d26-0cfa-444a-aeca-387531839971.png)

Run the following command to install etcdctl, the command-line client for interacting with etcd.

```bash
sudo apt update && sudo apt install etcd-client -y
```

### Install trivy

[Trivy](https://trivy.dev/latest/) is a vulnerability scanner for containers and other artifacts. While typically used in CI/CD pipelines rather than directly on nodes, you'll install it on the control node to become familiar with the tool.

Run the following commands to install the Trivy.

```bash
sudo apt install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy
```

**Reference:** [https://trivy.dev/latest/getting-started/](https://trivy.dev/latest/getting-started/)

## Install Ingress Controller

Running an ingress controller in a local Kubernetes cluster can be challenging since it requires a Service of type LoadBalancer which isn't available in a local environment. However, you can use [MetalLB](https://metallb.io/) which is a load balancer implementation for bare metal Kubernetes clusters.

### Configure kube-proxy

Run the following commands to edit the [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) configuration to use IPVS mode and enable strict ARP for MetalLB.

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e 's/mode: ""/mode: "ipvs"/' -e 's/strictARP: false/strictARP: true/' | \
kubectl apply -f - -n kube-system
```

### Install MetalLB

Run the following command to add the MetalLB repository and install MetalLB.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

Run the following command to create a MetalLB configuration.

```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.93.10/32 # this is the IP address of the control node
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
EOF
```

!!! danger
    Make sure you update the IP address in the configuration to match the IP address of the control node.

!!! tip
    If you get an error message like the following:
    
    ```
    Error from server (InternalError): error when creating "STDIN": Internal error occurred: failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": failed to call webhook: Post "https://metallb-webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-ipaddresspool?timeout=10s": dial tcp 10.104.133.116:443: connect: connection refused
    Error from server (InternalError): error when creating "STDIN": Internal error occurred: failed calling webhook "l2advertisementvalidationwebhook.metallb.io": failed to call webhook: Post "https://metallb-webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-l2advertisement?timeout=10s": dial tcp 10.104.133.116:443: connect: connection refused
    ``` 
  
    Simply wait a few seconds and try again.

This is equivalent to simply using [NodePort](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters) services but will mimic the behavior of a LoadBalancer service and assign the ingress service an IP address.

**Reference:** [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)

### Install Ingress Controller

Ingress Controllers provides Layer 7 load balancing for HTTP, HTTPS, and TCP traffic, allowing external access to services within your Kubernetes cluster.

Run the following command to install the Ingress-Nginx Controller.

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

**Reference:** [https://kubernetes.github.io/ingress-nginx/deploy/#quick-start](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)

## Take a snapshot

At this point, it's recommended to take another snapshot of each virtual machine. Give it a descriptive name like **after-k8s-install** so you can easily revert back to this point if you break something in your cluster.