## Configure the control and worker nodes

For each of the nodes (control and workers), you will need to install the necessary packages and configure the system. Each node needs to have containerd, crictl, kubeadm, kubelet, and kubectl installed.

Start by SSH'ing into each node and make sure you are running as root.

```bash
sudo -i
```

### System Updates

To avoid Kubernetes data such as contents of Secret object being written to tmpfs, "swap to disk" must be disabled. Additionally see [@thokin's comment](https://discuss.kubernetes.io/t/swap-off-why-is-it-necessary/6879) and the [kubeadm prerequisites](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) to understand why swap must be disabled.

**Reference:** [https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory](https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory)

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Update the system and install necessary packages.

```bash
apt-get update && apt-get upgrade -y
```

### Install `containerd`

Kubernetes uses the Container Runtime Interface (CRI) to interact with container runtimes and **containerd** is the container runtime that Kubernetes uses (it was dockerd in the past).

We will need to install containerd on each node from the Docker repository and configure it to use systemd as the cgroup driver.

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

Configure containerd to use systemd as the cgroup driver and use systemd cgroups.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

Update containerd to load the overlay and br_netfilter modules.

```bash
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Update kernel network settings to allow traffic to be forwarded.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional)

```bash
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Load the kernel modules and apply the sysctl settings to ensure they changes are used by the current system.

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

- kubeadm: the command to bootstrap the cluster.
- kubelet: the component that runs on all of the machines in the cluster and does things like starting pods and containers.
- kubectl: the command line tool to interact with the cluster.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

Add the Kubernetes repository.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

Update the system, install the Kubernetes packages, and lock the versions so they don't get unintentionally updated.

```bash
apt-get update
apt-get install -y kubelet=1.30.3-1.1 kubeadm=1.30.3-1.1 kubectl=1.30.3-1.1
apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet.

```bash
systemctl enable --now kubelet
```

### Install critctl

CRI-O is an implementation of the Container Runtime Interface (CRI) used by the kubelet to interact with container runtimes.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o)

Install crictl by downloading the binary to the system.

```bash
export CRICTL_VERSION="v1.30.1"
export CRICTL_ARCH=$(dpkg --print-architecture)
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$CRICTL_VERSION/crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz
tar zxvf crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz -C /usr/local/bin
rm -f crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz
```

Verify crictl is installed.

```bash
crictl version
```

### Configure crictl

Configure crictl to use the containerd socket. This is because default settings for crictl have been deprecated.

```sh
crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock
```

> If you don't do this, you will see the following error when you run `crictl ps`.

```text
WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
WARN[0000] image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
```

Reference: https://github.com/kubernetes-sigs/cri-tools/issues/868#issuecomment-1926494368

> ⛔️ Jump back to the [Configure the control and worker nodes](#configure-the-control-and-worker-nodes) section and repeat these steps for each worker node before proceeding.

## Install kubernetes on control node

The control node is where the Kubernetes control plane components will be installed. This includes the API server, controller manager, scheduler, and etcd. The control node will also run the CNI plugin to provide networking for the cluster.

### Control plane installation with kubeadm

Using kubeadm, install the Kubernetes with the `kubeadm init` command. This will install the control plane components and create the necessary configuration files.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

```bash
kubeadm init --kubernetes-version 1.30.3 --pod-network-cidr 192.168.0.0/16 --v=5
```

Export the kubeconfig file so the root user can access the cluster.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### CNI plugin installation with Cilium CLI

Cilium will be used as the CNI plugin for the cluster. Cilium is a powerful, efficient, and secure CNI plugin that provides network security, observability, and troubleshooting capabilities.

**Reference:** [https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

Install the Cilium CLI.

> The Cilium CLI version will be pinned to `v0.16.24`. To get the latest version, visit the [Cilium CLI releases page](https://github.com/cilium/cilium-cli/releases) and update the version number.

```bash
export CILIUM_VERSION="v0.16.24"
export CILIUM_ARCH=$(dpkg --print-architecture)

# Download the Cilium CLI binary and its sha256sum
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_VERSION}/cilium-linux-${CILIUM_ARCH}.tar.gz{,.sha256sum}

# Verify sha256sum
sha256sum --check cilium-linux-${CILIUM_ARCH}.tar.gz.sha256sum

# Move binary to correct location and remove tarball
tar xzvf cilium-linux-${CILIUM_ARCH}.tar.gz -C /usr/local/bin
rm cilium-linux-${CILIUM_ARCH}.tar.gz{,.sha256sum}
```

Verify the Cilium CLI is installed.

```bash
cilium version --client
```

Install Cilium.

```bash
cilium install --version 1.17.1
```

Wait for the CNI plugin to be installed.

```bash
cilium status --wait
```

After a few minutes, you should see output like this which shows the Cilium CNI plugin is installed and running.

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

Exit the root shell and run the following commands to configure kubectl for your normal user account.

```bash
exit
```

Configure kubectl to connect to the cluster.

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

## Join the worker nodes to the cluster

Log into each worker node, make sure you are in the root shell, and paste and run the join command that you copied from the control node.

**Reference:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)

> If your shell does not show `root@worker`, you can run `sudo -i` to switch to the root user.

Run a command similar to the following on each worker node.

```bash
kubeadm join 192.168.120.130:6443 --token xxxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxx
```

After the worker nodes have joined the cluster, log back into the control node and verify the nodes are listed as **Ready**.

```bash
kubectl get nodes -w
```

> At this point you probably don't need to log into the worker nodes again. You can work with cluster from the control node.

## Configure kubectl and tools

Log back into the control node as your normal user and configure kubectl. These configurations are optional but will make your life easier when working with Kubernetes.

**Reference:** [https://kubernetes.io/docs/reference/kubectl/quick-reference/](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

Install kubectl completion

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

Set kubectl alias so you can use `k` instead of `kubectl`.

```bash
cat <<EOF | tee -a ~/.bashrc
alias k=kubectl
complete -o default -F __start_kubectl k
EOF
```

Reload the bash profile.

```bash
source ~/.bashrc
```

Install jq to help with formatting output and strace for debugging.

```bash
sudo apt-get install jq strace -y
```

Install helm to help with installing applications.

**Reference:**[https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm ./get_helm.sh
```

Install the etcdctl tool to interact with the etcd database.

```bash
sudo apt install etcd-client -y
```

Configure .vimrc to help with editing YAML files.

```bash
cat <<EOF | tee -a ~/.vimrc
set tabstop=2
set expandtab
set shiftwidth=2
EOF
```
