After you get passed the CKAD exam, you should have a solid understanding of fundamental Kubernetes concepts and be ready to move on to the next level; that is, being a Kubernetes administrator. This exam, is just as intense as the CKAD exam but focuses on the administrative tasks that you would need to perform as a Kubernetes administrator. This includes things like setting up and managing clusters, managing workloads, taking backups of the etcd database, and more.

Take a look at the [Certified Kubernetes Administrator (CKA)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) page for Domain and Competency details.

Some of the hands on activities that you should be comfortable with are:

- Upgrading a cluster
- Managing role-based access control (RBAC)
- Managing network policies
- Managing storage
- Managing cluster components
- etcd database backup and restore
- Configuring the kube-apiserver
- Troubleshooting a cluster

Start by SSH'ing into the control node and running the following command to get sudo access:

```bash
sudo -i
```

## Managing core Kubernetes components

The core components of a Kubernetes cluster are:
- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler

When using kubeadm, these components are automatically deployed as static pods. Static pods are managed by the kubelet and are defined in a manifest file located at `/etc/kubernetes/manifests` on the control plane node. The kubelet will automatically start and manage these pods.

Run the following command to view the manifest files.

```bash
ll /etc/kubernetes/manifests
```

To make changes to any of these components, you will need to edit the manifest file and the kubelet will automatically restart the pod with the new configuration.

!!! tip
    When making changes to the manifest files, you should always make a backup of the original file before making any changes. This will allow you to revert back to the original configuration if something goes wrong.

After making changes to the manifest file, you can check the status of the components by running the following command.

```bash
watch crictl ps
```

You might encounter a situation where the components are not running. In this case, you should check the status of the kubelet process by running the following command.

```bash
systemctl status kubelet
```

If you see `active (running)` highlighted in green, then the kubelet is running. If not, you will need to start the kubelet by running the following command.

```bash
systemctl start kubelet
```

You might also encounter a situation where the kubelet is running but some components are not running. You can check the status of the components by running the following command.

```bash
crictl ps
```

You should see the following output.

```text
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
88639aaca1dd1       d48f992a22722       2 days ago          Running             kube-scheduler            18                  ec844d5e5a039       kube-scheduler-control
2c7467d60a7a6       014faa467e297       3 days ago          Running             etcd                      7                   f5c73fde48d08       etcd-control
cdfa1217a1a25       8e97cdb19e7cc       3 days ago          Running             kube-controller-manager   17                  f78f21be20d53       kube-controller-manager-control
0930536d04d9a       528878d09e0ce       4 weeks ago         Running             cilium-envoy              6                   077c262fdc09c       cilium-envoy-9r7dk
c6dcda213517c       2351f570ed0ea       4 weeks ago         Running             kube-proxy                6                   b4fbd18b3b311       kube-proxy-ffd5z
```

If see some components are not running, you should look to the logs of the kubelet to see what is going on. You can do this by running the following command.

```bash
journalctl -u kubelet
```

## Upgrading a cluster

Upgrading a cluster is a critical task that needs to be done carefully to avoid any downtime. The upgrade process involves upgrading the control plane components and then upgrading the worker nodes. It's a tedious and time consuming task on the exam, but worth a lot of points and can be done quickly with practice.

### Upgrade the control plane

Run the command to get the list of nodes and the Kubernetes version they are running.

```bash
kubectl get nodes
```

!!! note
    If you are running as the root user, you will need to run the following command to export the KUBECONFIG environment variable.

    ```bash
    export KUBECONFIG=/etc/kubernetes/admin.conf
    ```

Before upgrading the control plane, you should drain the control node to prevent any new Pods from being scheduled on it.

```bash
kubectl drain control --ignore-daemonsets
```

Check to see what version of the kubelet you are running.

```bash
kubelet --version
```

Check to see what version of kubeadm you are running.

```bash
kubeadm version
```

Both the kubelet and kubeadm should be running v1.31.6. Change the package repository to point to the v1.32.x packages.

```bash
sed -i 's/v1.31/v1.32/g' /etc/apt/sources.list.d/kubernetes.list
```

Update the package list.

```bash
apt update
```

**Reference:** https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used


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

Hold the kubeadm package to prevent automatic upgrades again.

```bash
apt-mark hold kubeadm
```

Wait a few minutes and run the following command to see if the control plane components are running.

```bash
kubeadm upgrade plan
```

!!! note
    If you see any preflight errors, you may be able to simply re-run the command as they are often transient and resolve themselves.

If the upgrade check went well, you will see the following output.

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

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
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

Unhold the kubelet and kubectl packages so they can be upgraded.

```bash
apt-mark unhold kubelet kubectl
```

Upgrade the kubelet and kubectl packages to v1.32.2.

```bash
apt install kubelet=1.32.2-1.1 kubectl=1.32.2-1.1
```

Hold the kubelet and kubectl packages to prevent automatic upgrades again.

```bash
apt-mark hold kubelet kubectl
```

With the control plane upgraded, you can uncordon the control node to allow new Pods to be scheduled on it again.

```bash
kubectl uncordon control
```

!!! note
    You might need to wait a minute or two for the control plane components to come back up since the kubelet will need to restart and eventually restart the static pods.

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

Before upgrading the worker nodes, you should drain the worker nodes to prevent any new Pods from being scheduled on them.

```bash
kubectl drain worker-1 --ignore-daemonsets
```


### Upgrade the worker nodes

Start by SSH'ing into the worker node and run the following command to get sudo access.

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

Upgrade the worker node by running the following command.

```bash
kubeadm upgrade node --v=5
```

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

Restart the kubelet service.

```bash
service kubelet restart
```

Wait a minute or two and check to make sure the kubelet service is running.

```bash
service kubelet status
```

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

Uncordon the worker node to allow new Pods to be scheduled on it.

```bash
kubectl uncordon worker-1
```

!!! danger
    Drain the next worker node and repeat the above steps for the remaining worker nodes.

**Reference:** https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

## Backing up and restoring etcd

The etcd database is a key component of a Kubernetes cluster. It stores the state of the cluster and is used by the control plane components to coordinate and manage the cluster. Backing up the etcd database is critical to ensure that you can recover from any data loss or corruption.

### Take a snapshot of etcd

The etcdctl is the tool you will use to manage the etcd database. It is installed on the control plane node but needs sudo access to run.

Run the following command to get sudo access.

```bash
sudo -i
```

Now run the following command to see the available commands.

```bash
etcdctl --help # or -h
```

You will see there is a command to take a snapshot of the etcd database. Let's see what the command needs.

```bash
etcdctl snapshot save -h
```

You will see the command needs a path to save the snapshot to and accepts a few flags to configure how the client connects to the etcd server.

The etcd database is run as a static pod on the control plane node. Static Pods are defined in a manifest file located at `/etc/kubernetes/manifests/etcd.yaml`. This file will give you the information you need to connect to the etcd server.

You can view the manifest file by running the following command.

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

It's a fairly long file, but locate the `- command` section. You will see the following.

```yaml
- command:
    - etcd
    - --advertise-client-urls=https://172.16.25.132:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://172.16.25.132:2380
    - --initial-cluster=control=https://172.16.25.132:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://172.16.25.132:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://172.16.25.132:2380
    - --name=control
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

Based on what we see here, it looks like we can connect to the etcd server using the same certificates that are used by the Pod. The `--cert-file` and `--key-file` flags specify the client certificate and key to use for authentication. The `--trusted-ca-file` flag specifies the CA certificate to use for verifying the server's certificate.

Run the following command to take a snapshot of the etcd database.

```bash
etcdctl snapshot save /tmp/etcd-backup.db \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```

### Restore etcd from a snapshot

With the snapshot taken, let's see how to restore the etcd database from a snapshot.

First thing you need to do is stop the etcd Pod. You can do this by simply moving the etcd manifest file to a different location. The kubelet will see that the manifest file is no longer there and stop the Pod.

```bash
cd /etc/kubernetes/manifests

# move all manifest files to parent directory
mv * .. 
```

Make sure the control plane components are not running.

```bash
watch crictl ps
```

Once you see that the etcd Pod is not running, press `Ctrl+C` to exit the watch command and proceed to restore the etcd database from the snapshot. 

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

Now that you have restored the etcd database to a different directory, you will need to update the etcd static pod manifest file to point to the new data directory.

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

## Managing cluster components

The cluster components are the core components of a Kubernetes cluster. They are responsible for managing the cluster and ensuring that everything is running as expected.

In Kubernetes clusters that have been setup using kubeadm, all control plane components are run as static pods with manifest files located at `/etc/kubernetes/manifests` and managed by the kubelet.

The status of the static pods can be checked by running the following command.

```bash
crictl ps
```

### Managing kubelet

The kubelet runs as a process on each node in the cluster and is responsible for managing the Pods and containers. 

Run the following command to view the status of the kubelet process.

```bash
systemctl status kubelet
```

You should see `active (running)` highlighted in green. If not, you can start the kubelet by running the following command.

```bash
systemctl start kubelet
```

If the kubelet is not starting for whatever reason, you can check the logs by running the following command.

```bash
journalctl -u kubelet
```

To view how the kubelet is configured, you can run the following command.

```bash
ps aux | grep kubelet
```

The text will be a bit jumbled, but you should be able to see the flags that are being passed to the kubelet process.

For example, you might see something like this.

```text
root        9848  1.0  2.5 2044616 100760 ?      Ssl  13:37   0:37 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.k8s.io/pause:3.10
```

This shows that the kubelet is using a kubeconfig file located at `/etc/kubernetes/kubelet.conf` and a configuration file located at `/var/lib/kubelet/config.yaml`.

### Managing kube-apiserver

The kube-apiserver is the control plane component that exposes the Kubernetes API. It is the front-end for the Kubernetes control plane. So securing the kube-apiserver is critical to the security of the entire cluster.

The kube-apiserver is configured using a configuration file located at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

There are quite a few flags that you can use to configure the kube-apiserver. Be sure to check the reference link below for a complete list of flags.

Whenever you are asked to make changes to the way the kube-apiserver is accessed or secured, you will need to edit this file and the kubelet will automatically restart the kube-apiserver Pod with the new configuration.

Run the following command to view the kube-apiserver static pod manifest file.

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Run the following command to edit the kube-apiserver.yaml file to enable the [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) admission plugin.

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Locate the `--enable-admission-plugins` flag, add `ValidatingAdmissionPolicy` to the list of plugins, then save and exit the file.

!!! note
    The list of plugins is comma separated.

After making changes to the kube-apiserver static pod manifest file (or any other static pod manifest file), you should run the following command to ensure the kubelet has restarted the Pod with the new configuration.

```bash
watch crictl ps
```

You will see the kube-apiserver be removed from the list of running Pods and then reappear with the new configuration.

!!! tip
    If you don't see the kube-apiserver reappear, that probably means there was an error in the configuration file. So it is always a good idea to make a backup of the original file before making any changes.

**Reference:** [https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

## Securing a cluster

### RBAC and Service Accounts

In Kubernetes, there is no notion of users. Instead, identities can be represented as Service Accounts. Service Accounts are used to authenticate Pods to the API server and these service accounts can be bound to roles which define what actions can be performed on resources.

When you manage RBAC in Kubernetes, you define a Role or ClusterRole which is a set of permissions. You then create a RoleBinding or ClusterRoleBinding which binds the Role or ClusterRole to a Service Account. The difference between a Role and a ClusterRole is that a Role is namespaced and a ClusterRole is not. Likewise a RoleBinding is namespaced and a ClusterRoleBinding is not.

!!! note
    Roles can only be combined RoleBindings but a ClusterRole can be used with both RoleBindings and ClusterRoleBindings making it more flexible across namespaces.


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