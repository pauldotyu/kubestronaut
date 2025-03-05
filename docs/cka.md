## Overview

After you get past the CKAD exam, you should have a solid understanding of fundamental Kubernetes concepts and be ready to move on to the next level. This exam, is just as intense as the CKAD exam but focuses on the administrative tasks that you would need to perform as a Kubernetes administrator. This includes things like setting up and managing clusters, managing workloads, taking backups of the etcd database, performing cluster upgrades, and more.

Read through the [Certified Kubernetes Administrator (CKA)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) page for Domain and Competency details and information on how to register for the exam.

Some of the hands on activities that you should be comfortable with are:

- Upgrading a cluster
- Managing role-based access control (RBAC)
- Managing network policies
- Managing storage
- Managing cluster components
- etcd database backup and restore
- Configuring the kube-apiserver
- Troubleshooting a cluster

To begin, SSH into the control node and switch to the root user.

```bash
sudo -i
```

!!! tip
    Many CKA tasks require elevated permissions to perform certain tasks especially when working with files and directories that are owned by the root user or using the crictl CLI tool. If you need to run kubectl commands as the root user, you'll need to export the KUBECONFIG environment variable by running the following command:

    ```bash
    export KUBECONFIG=/etc/kubernetes/admin.conf
    ```

## Managing core components

The core components of a Kubernetes cluster are:

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler

When using [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) to bootstrap a cluster, these components are automatically deployed as static pods. [Static pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/) are managed by the kubelet and are defined in manifest files located at `/etc/kubernetes/manifests` on the control plane node. The kubelet will automatically start and manage these pods.

Run the following command to view the manifest files.

```bash
ll /etc/kubernetes/manifests
```

To make changes to any of these components, you'll need to edit the manifest file and the kubelet will automatically restart the pod with the new configuration.

!!! tip
    When making changes to the manifest files, you should always make a backup of the original file before making any changes. This will allow you to revert back to the original configuration if something goes wrong.

The status of the static pods can be checked by running the following command.

```bash
crictl ps
```

If all is well, you should see output similar to the following.

```text
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
3193217d296c0       f9818b4cd5fcb       8 hours ago         Running             frr-metrics               0                   c890c7a3a0205       metallb-speaker-v5b4s
b18b948ca4484       f9818b4cd5fcb       8 hours ago         Running             reloader                  0                   c890c7a3a0205       metallb-speaker-v5b4s
9d613f23e25d4       f9818b4cd5fcb       8 hours ago         Running             frr                       0                   c890c7a3a0205       metallb-speaker-v5b4s
5e1922db4e666       f1e519d0f31b3       8 hours ago         Running             speaker                   0                   c890c7a3a0205       metallb-speaker-v5b4s
6c3e8111d7b31       2f6c962e7b831       8 hours ago         Running             coredns                   0                   66036843831f6       coredns-7c65d6cfc9-cvjb8
62813ada99f50       2f6c962e7b831       8 hours ago         Running             coredns                   0                   c4b818a26a66a       coredns-7c65d6cfc9-vldm9
35463beb7567d       62788cf8a7b25       8 hours ago         Running             cilium-operator           0                   1b0127859d68d       cilium-operator-595959995b-qj6rh
1290ffeb922f3       00ec0a2d78b34       8 hours ago         Running             cilium-envoy              0                   5de31704930bb       cilium-envoy-f2f8h
89c01101f07bf       866c4ad7c1bbd       8 hours ago         Running             cilium-agent              0                   9d9cacb4d69ed       cilium-9fm7w
8cbedd9d34e91       dc056e81c1f77       8 hours ago         Running             kube-proxy                0                   278ea959179c6       kube-proxy-wbfgj
c8f1fa1747310       e0b799edb30ee       8 hours ago         Running             kube-scheduler            0                   caed0974f8975       kube-scheduler-control
9efee977a5534       27e3830e14027       8 hours ago         Running             etcd                      0                   c3e78bddfb219       etcd-control
e2184b59d426e       873e20495ccf3       8 hours ago         Running             kube-apiserver            0                   7ea6c543df23b       kube-apiserver-control
e1c292d2aa6be       389ff6452ae41       8 hours ago         Running             kube-controller-manager   0                   0ca20c7dfab9f       kube-controller-manager-control
```

### Managing kube-apiserver

The kube-apiserver is the control plane component that exposes the Kubernetes API. It is the front-end for the Kubernetes control plane. So securing the kube-apiserver is critical to the security of the entire cluster. Most of your tasks will involve making changes to the kube-apiserver configuration.

The kube-apiserver is configured using a manifest file located at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

Run the following command to view the kube-apiserver static pod manifest file.

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

!!! tip
    There are a lot of flags that you can use to configure the kube-apiserver. Be sure to bookmark the [kube-apiserver reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) for a complete list and explanation of each flag.

Whenever you are asked to make changes to the way the kube-apiserver is accessed or secured, this is the file you need to edit and the kubelet will automatically restart the kube-apiserver Pod with the new configuration.

#### Enable ValidatingAdmissionPolicy

Run the following command to edit the kube-apiserver manifest to enable the [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) admission plugin. It validates requests to the Kubernetes API before they are persisted to etcd. It can be used to enforce policies like ensuring that all Pods have resource requests and limits, or that all Pods have a certain label.

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Locate the `--enable-admission-plugins` flag, add `ValidatingAdmissionPolicy` to the list of plugins, then save and exit the file.

!!! note
    The list of plugins is comma separated.

After making changes to the kube-apiserver manifest you should keep a watch on the static Pods to ensure the kube-apiserver restarts with the new configuration. It is very common to make a mistake and not realize it until you try to use kubectl and it doesn't work.

Run the following command to watch the static Pods and ensure they are all running.

```bash
watch crictl ps
```

You will see the kube-apiserver be removed from the list of running Pods and then reappear with the new configuration.

!!! tip
    If you don't see the kube-apiserver reappear, that probably means there was an error in the configuration file. So it is always a good idea to make a backup of the original file before making any changes.

### Troubleshooting kubelet

The kubelet runs as a process on each node in the cluster and is responsible for managing the Pods and containers. 

You might encounter a situation where the components aren't running. In this case, you should check the status of the kubelet process by running the following command.

```bash
systemctl status kubelet
```

If you see `active (running)` highlighted in green, then the kubelet is running. If not, you'll need to start the kubelet by running the following command.

```bash
systemctl start kubelet
```

If the kubelet is running but you suspect some components aren't running. You can check the status of the components by running the following command.

```bash
crictl ps
```

You can see in the output that a few components aren't running, like the kube-apiserver and kube-controller-manager. This should lead you to check the manifests for these components to see what is going on or check the logs of the kubelet to see if there are any errors.

```text
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
88639aaca1dd1       d48f992a22722       2 days ago          Running             kube-scheduler            18                  ec844d5e5a039       kube-scheduler-control
2c7467d60a7a6       014faa467e297       3 days ago          Running             etcd                      7                   f5c73fde48d08       etcd-control
cdfa1217a1a25       8e97cdb19e7cc       3 days ago          Running             kube-controller-manager   17                  f78f21be20d53       kube-controller-manager-control
0930536d04d9a       528878d09e0ce       4 weeks ago         Running             cilium-envoy              6                   077c262fdc09c       cilium-envoy-9r7dk
c6dcda213517c       2351f570ed0ea       4 weeks ago         Running             kube-proxy                6                   b4fbd18b3b311       kube-proxy-ffd5z
```

To help troubleshoot the kubelet, you can check the logs by running the following command.

```bash
journalctl -u kubelet
```

If you need to check to see how the kubelet is configured, you can run the following command.

```bash
ps aux | grep kubelet
```

!!! note
    This assumes the kubelet is running as a process on the node.

The text will be a bit jumbled, but you should be able to see the flags that are being passed to the kubelet process.

For example, you might see something like this.

```text
root        9848  1.0  2.5 2044616 100760 ?      Ssl  13:37   0:37 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.10
```

This shows that the kubelet is using a configuration files located at `/etc/kubernetes/kubelet.conf` and `/var/lib/kubelet/config.yaml` which can also be helpful when troubleshooting.

### Backing up and restoring etcd

The etcd database is a key component of a Kubernetes cluster. It stores the state of the cluster and is used by the control plane components to coordinate and manage the cluster. Backing up the etcd database is critical to ensure that you can recover from any data loss or corruption.

#### Take a snapshot of etcd

The etcdctl is the tool you'll use to manage the etcd database. It is installed on the control plane node but needs sudo access to run.

Now run the following command to see the available commands.

```bash
etcdctl --help # or -h
```

!!! warning
    Make sure you've switched to the root user with `sudo -i` before running the above command.

You will see there is a command to take a snapshot of the etcd database. Let's see what the command needs.

```bash
etcdctl snapshot save -h
```

The command needs a path to save the snapshot to and accepts a few flags to configure how the client connects to the etcd server.

With kubeadm, etcd is run as a static pod on the control plane node and defined in a manifest file located at `/etc/kubernetes/manifests/etcd.yaml`. This file will give you everything you need to connect to the etcd server.

You can view the manifest file by running the following command.

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

It's a fairly long file, but locate the `- command` section and you'll see the following:

```yaml
- command:
    - etcd
    - --advertise-client-urls=https://172.16.25.132:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt           # need this
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://172.16.25.132:2380
    - --initial-cluster=control=https://172.16.25.132:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key            # need this
    - --listen-client-urls=https://127.0.0.1:2379,https://172.16.25.132:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://172.16.25.132:2380
    - --name=control
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt         # need this
```

Based on what we see here, it looks like we can connect to the etcd server using the same certificates that are used by the Pod. The `--cert-file` and `--key-file` flags specify the client certificate and key to use for authentication. The `--trusted-ca-file` flag specifies the CA certificate to use for verifying the server's certificate.

Using the certificates that the etcd pod uses, run the following command to take a snapshot of the etcd database and save it to `/tmp/etcd-backup.db`.

```bash
etcdctl snapshot save /tmp/etcd-backup.db \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```

#### Restore etcd from a snapshot

With the snapshot saved, let's see how to restore the etcd database from a snapshot.

First thing you need to do is stop the etcd Pod. You can do this by simply moving the etcd manifest file to a different location. The kubelet will see that the manifest file is no longer there and stop the Pod.

```bash
cd /etc/kubernetes/manifests

# move all manifest files to parent directory
mv * .. 
```

Make sure the control plane components aren't running.

```bash
watch crictl ps
```

Once you see that the etcd Pod isn't running, press `Ctrl+C` to exit the watch command and proceed to restore the etcd database from the snapshot. 

Let's run help on the `etcdctl snapshot restore` command to see what it needs.

```bash
etcdctl snapshot restore -h
```

You will see that the command needs a path to the snapshot file, a path to the data directory which is where the etcd database is stored, and a few flags to configure how the client connects to the etcd server, like the `etcdctl snapshot save` command.

The `--data-dir` flag is the path to the data directory where the etcd database is stored. This is usually `/var/lib/etcd` but you can confirm this by looking at the `- command` section of the etcd static pod manifest file.

When you perform a restore, you can specify a different data directory to restore to. This is useful if you want to keep the original data directory intact.

Run the following command to restore the etcd database from the snapshot.

```bash
etcdctl snapshot restore /tmp/etcd-backup.db \
--data-dir=/var/lib/etcd-from-backup \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```

Now that you've restored the etcd database to a different directory, you'll need to update the etcd static pod manifest file to point to the new data directory.

Make a backup of the original etcd static pod manifest file.

```bash
cp ../etcd.yaml etcd.yaml.bak
```

Use Vim to edit the etcd static pod manifest file.

```bash
vim ../etcd.yaml
```

Locate the `- hostPath` spec under `volumes` for `etcd-data` and update the path to the new data directory.

```yaml
- hostPath:
    path: /var/lib/etcd-from-backup
    type: DirectoryOrCreate
  name: etcd-data
```

Save and exit the file.

Move the static pod manifest files back to the `/etc/kubernetes/manifests` directory.

```bash
# move all yaml files back
mv ../*.yaml .
```

Run the following command and wait for the etcd Pod to start.

```bash
watch crictl ps
```

**Reference:** [https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)

## Securing a cluster

We'll get to security a bit more in the next section but one component of cluster security is RBAC and ServiceAccounts. In Kubernetes, users aren't stored as API objects but are authenticated through various mechanisms (certificates, tokens, OIDC, etc.). Inside the cluster, workloads use [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/) for identity, which are Kubernetes API objects that can be bound to roles which define what actions can be performed on resources.

When you manage RBAC in Kubernetes, you define a [Role or ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) which is a set of permissions. You then create a [RoleBinding or ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) which binds the Role or ClusterRole to a Service Account, user, or group. The difference between a Role and a ClusterRole is that a Role is namespaced and a ClusterRole isn't. Likewise a RoleBinding is namespaced and a ClusterRoleBinding isn't.

!!! note
    Roles can only be combined with RoleBindings but a ClusterRole can be used with both RoleBindings and ClusterRoleBindings making it more flexible across namespaces.

### Granting RBAC to ServiceAccounts

Run the following command to create a new Service Account.

```bash
kubectl create serviceaccount my-service-account
``` 

Run the following command to create a new Role.

```bash
kubectl create role my-role --verb=get --verb=list --resource=pods
```

Run the following command to bind the Role to the Service Account.

```bash
kubectl create rolebinding my-role-binding --role=my-role --serviceaccount=default:my-service-account
```

Run the following command to verify the service account can only list pods.

```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:my-service-account
```

You should see `yes` as the output.

Run the following command to verify the service account cannot create pods.

```bash
kubectl auth can-i create pods --as=system:serviceaccount:default:my-service-account
```

You should see `no` as the output.

**Reference:** [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

## Upgrading a cluster

Upgrading a cluster is a critical task that needs to be done carefully to avoid any downtime. The upgrade process involves upgrading the control plane components and then upgrading the worker nodes. It's a tedious and time consuming task on the exam, but worth a lot of points and can be done quickly with practice.

### Control plane

Run the command to get the list of nodes and the Kubernetes version they are running.

```bash
kubectl get nodes
```

!!! warning
    When running as the root user, you'll need to run the following command to export the KUBECONFIG environment variable to run kubectl commands.

    ```bash
    export KUBECONFIG=/etc/kubernetes/admin.conf
    ```

#### Drain node

Before upgrading the control plane, you should drain the control node to prevent any new Pods from being scheduled on it.

```bash
kubectl drain control --ignore-daemonsets
```

#### Package repository updates

Check to see what version of kubeadm and kubelet you are running.

```bash
kubeadm version
kubelet --version
```

Both kubeadm and kubelet should be running v1.31.6. Change the package repository to point to the v1.32.x packages.

```bash
sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
```

Update the package list.

```bash
apt update
```

**Reference:** [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used)

Check to see what versions of the Kubernetes packages you can upgrade to.

```bash
apt list --upgradable | grep kube
```

The packages were held back when you installed them to prevent automatic upgrades. Unhold the packages by running the following command.

```bash
apt-mark unhold kubeadm
```

Upgrade the kubeadm package to v1.32.2.

```bash
apt install -y kubeadm=1.32.2-1.1
```

Hold the kubeadm package again to prevent inadvertent upgrades.

```bash
apt-mark hold kubeadm
```

#### kubeadm upgrade

Run the following command to check if the upgrade is possible.

```bash
kubeadm upgrade plan
```

!!! note
    If you see any preflight errors, you may be able to simply re-run the command as they are often transient and resolve themselves.

If the upgrade check went well, you'll see the following output.

```text
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/config] Use 'kubeadm init phase upload-config --config your-config.yaml' to re-upload it.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.31.6
[upgrade/versions] kubeadm version: v1.32.2
[upgrade/versions] Target version: v1.32.2
[upgrade/versions] Latest version in the v1.31 series: v1.31.6

Components that must be upgraded manually after you've upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE       CURRENT   TARGET
kubelet     worker-1   v1.31.6   v1.32.2
kubelet     worker-2   v1.31.6   v1.32.2
kubelet     control    v1.32.2   v1.32.2

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            control   v1.31.6    v1.32.2
kube-controller-manager   control   v1.31.6    v1.32.2
kube-scheduler            control   v1.31.6    v1.32.2
kube-proxy                          1.31.6     v1.32.2
CoreDNS                             v1.11.3    v1.11.3
etcd                      control   3.5.15-0   3.5.16-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.32.2

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

Upgrade the control plane components by running the following command.

```bash
kubeadm upgrade apply v1.32.2 --v=5
```

When asked if you are sure you want to proceed, type `y` and press `Enter`.

!!! warning
    The upgrade process can take several minutes to complete.

#### kubelet and kubectl upgrades

When the control plane upgrade is complete, you can move on to upgrading the kubelet and kubectl. Unhold the kubelet and kubectl packages so they can be upgraded.

```bash
apt-mark unhold kubelet kubectl
```

Upgrade the kubelet and kubectl packages to v1.32.2.

```bash
apt install kubelet=1.32.2-1.1 kubectl=1.32.2-1.1
```

Hold the kubelet and kubectl packages again to prevent inadvertent upgrades.

```bash
apt-mark hold kubelet kubectl
```

#### Uncordon node

With the control plane upgraded, you can uncordon the control node to allow new Pods to be scheduled on it again.

```bash
kubectl uncordon control
```

!!! note
    You might need to wait a minute or two for the control plane components to come back up since the kubelet will need to restart and eventually restart the static pods.

#### Verify upgrade

Run the following command to verify the control node is upgraded to v1.32.2.

```bash
kubectl get nodes
```

Your output should look like this.

```text
NAME       STATUS   ROLES           AGE   VERSION
control    Ready    control-plane   15h   v1.32.2  # upgraded
worker-1   Ready    <none>          15h   v1.31.6
worker-2   Ready    <none>          15h   v1.31.6
```

You can see the control node is now running v1.32.2 and the worker nodes are still running v1.31.6. You need to upgrade the worker nodes next.

### Worker nodes

#### Drain node

Before upgrading the worker nodes, you should drain the worker nodes to prevent any new Pods from being scheduled on them.

```bash
kubectl drain worker-1 --ignore-daemonsets
```

!!! warning
    Make sure you are back on the control node before running the above command.

#### Package repository updates

SSH into the worker node and switch to the root user.

```bash
sudo -i
```

Update the package repository to point to the v1.32.x packages.

```bash
sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
```

Update the package list.

```bash
apt update
```

Unhold the kubeadm package so it can be upgraded.

```bash
apt-mark unhold kubeadm
```

Upgrade the kubeadm package to v1.32.2.

```bash
apt install kubeadm=1.32.2-1.1
```

Hold the kubeadm package to prevent automatic upgrades again.

```bash
apt-mark hold kubeadm
```

#### kubeadm upgrade

Upgrade the worker node by running the following command.

```bash
kubeadm upgrade node --v=5
```

#### kubelet and kubectl upgrades

Unhold the kubelet and kubectl packages so they can be upgraded.

```bash
apt-mark unhold kubectl kubelet
```

Upgrade the kubelet and kubectl packages to v1.32.2.

```bash
apt install kubelet=1.32.2-1.1 kubectl=1.32.2-1.1
```

Hold the kubelet and kubectl packages to prevent automatic upgrades again.

```bash
apt-mark hold kubectl kubelet
```

#### Restart kubelet

Restart the kubelet service.

```bash
service kubelet restart
```

Wait a minute or two and check to make sure the kubelet service is running.

```bash
service kubelet status
```

#### Verify upgrade

Back in the control node, run the following command to see if the worker node is ready.

```bash
kubectl get nodes
```

!!! warning
    Make sure you are back on the control node before running the above command.


Now you should see a worker node running v1.32.2.

```text
NAME       STATUS                     ROLES           AGE   VERSION
control    Ready                      control-plane   15h   v1.32.2 # upgraded
worker-1   Ready,SchedulingDisabled   <none>          15h   v1.32.2 # upgraded
worker-2   Ready                      <none>          15h   v1.31.6
```

#### Uncordon node

Uncordon the worker node to allow new Pods to be scheduled on it.

```bash
kubectl uncordon worker-1
```

!!! danger
    Drain the next worker node and repeat the above steps for the remaining worker nodes.

**Reference:** [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

## Additional Resources

- [CKA Exam Curriculum](https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.32.pdf)
- [CKA Certification Learning Path](https://kodekloud.com/learning-path/cka)
