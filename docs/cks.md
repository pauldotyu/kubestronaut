The last exam in the path to become a Kubestronaut is the CKS exam. This exam is probably the most difficult exam to clear. You will be tested on your ability to secure Kubernetes clusters and workloads. This includes things like setting up network policies, securing the etcd database, using external security tools, and more. The exam has evolved and recently been updated to include more emphasis on cluster setup and reviewing security configurations.

Take a look at the [Certified Kubernetes Security Specialist (CKS)](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist-cks-2/) page for Domain and Competency details.

Some of the hands on activities that you should be comfortable with are:

- Using kernel hardening tools like AppArmor and seccomp
- Using Falco to monitor runtime security
- Using CIS benchmarks to secure a cluster
- Performing static analysis of workloads and container images
- Implementing end-to-end traffic encryption with Cilium
- Using network policies to restrict traffic to applications
- Using Pod security standards to secure workloads 
- Securing the kube-apiserver

## Network Policies

I can't stress this enough, being proficient with network policies is critical to passing all the Kubernetes exams. Network policies are used to control the flow of traffic to and from pods. They are a powerful tool to secure your workloads and are a key component of securing a Kubernetes cluster.

### Cilium Network Policies

It is worth noting that in addition to standard Kubernetes network policies, Cilium also provides network policies that can be used to secure network traffic in a Kubernetes cluster. 

Cilium is a powerful networking and security tool that can be used to secure your Kubernetes cluster. It provides a number of features including network policies, encryption, and more.

Run the following command to create a Cilium network policy that allows traffic to nginx1 from nginx2 but denies all traffic to nginx2.

```bash
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: deny-all
spec:
  endpointSelector:
    matchLabels:
      app: nginx2
  ingress:
  - fromEndpoints: []
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-nginx1-to-nginx2
spec:
  endpointSelector:
    matchLabels:
      app: nginx1
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: nginx2
EOF
```

Create the nginx1 and nginx2 pods.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  labels:
    app: nginx1
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx1
spec:
  selector:
    app: nginx1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  labels:
    app: nginx2
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

Test the network policy by trying to access the nginx2 pod from the nginx1 pod.

```bash
kubectl exec -it nginx1 -- curl nginx2 --connect-timeout 1
```

You will see that the connection was not allowed.

Now, try to access the nginx1 pod from the nginx2 pod.

```bash
kubectl exec -it nginx2 -- curl nginx1
```

You will see that the connection was successful.

CiliumNetworkPolicy may not appear on the CKS exam, but it is a powerful tool that gives you additional control over network traffic in your cluster like applying DNS-based or layer 7 policies.

Be sure to check out the Cilium documentation linked before for more information.

**Reference:** [https://docs.cilium.io/en/stable/security/policy/](https://docs.cilium.io/en/stable/security/policy/)

### Encrypting Pod Traffic with Cilium

Cilium can be used to encrypt pod traffic in a Kubernetes cluster. This is done by enabling encryption in the Cilium configuration.

There are two types of encryption that can be used:

- WireGuard - a fast and secure VPN protocol
- IPsec - a secure network protocol suite that authenticates and encrypts the packets of data sent over a network

By default encryption is disabled in Cilium. You can confirm using this command.

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium encrypt status
```

To enable encryption in Cilium with Wireguard on install, you can run the following command.

```bash
cilium install --version 1.17.1 \
--set encryption.enabled=true \
--set encryption.type=wireguard
```

Since we initially installed Cilium without encryption, we can enable it by running the following command.

```bash
cilium config set enable-wireguard true
```

You can confirm that encryption is enabled by running the following command.

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium encrypt status
```

You should now see that encryption is enabled using the Wireguard protocol and the interface is `cilium_wg0`.

```text
Encryption: Wireguard                 
Interface: cilium_wg0
        Public key: 5qmou/aktRjAAxocap8M+NNa5guOPRCubqqMxIa5v1A=
        Number of peers: 2
```

To verify traffic is being sent through the encrypted interface, you can run the following commands to install and run tcpdump on the cilium_wg0 interface.

```bash
apt-get update
apt-get -y install tcpdump
tcpdump -n -i cilium_wg0
```

Be sure to check out the Cilium documentation linked below for more information on how to configure encryption in Cilium.

**Reference:** [https://docs.cilium.io/en/stable/security/network/encryption/](https://docs.cilium.io/en/stable/security/network/encryption/)

## Pod Security Standards

Pod security standards are a set of best practices that you can use to secure your workloads. These standards are enforced by the kube-apiserver and can be used to ensure that your workloads are secure.

Enabling Pod Security Standards is done by labeling a namespace with a label. This label can be set to various values to enforce different levels of security. The values are:

- Privileged - provides the most flexibility and is the least secure
- Baseline - provides minimal security
- Restricted - provides the most security

**Reference:** [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/) 


### Enabling Pod Security Standards


Run the following command to label the default namespace with the `pod-security.kubernetes.io/enforce` label set to `enforce`.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: n297
  labels:
    pod-security.kubernetes.io/enforce: restricted # add
EOF
``` 

Create a pod that violates the Pod Security Standards.

```bash
kubectl apply -n n297 -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80
EOF
```

You will see the pod was not created because the pod can allow privilege escalation.

```text
Error from server (Forbidden): error when creating "STDIN": pods "nginx" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

To fix the issue, update the pod spec to meet the Pod Security Standards.

```bash
kubectl apply -n n297 -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80
      securityContext:                      
        allowPrivilegeEscalation: false   # disallow privilege escalation
        capabilities:                     # drop all capabilities
          drop:
            - ALL
        runAsNonRoot: true                # disallow running as root
        seccompProfile:
          type: RuntimeDefault            # use the default seccomp profile
EOF
```

**Reference:** [https://kubernetes.io/docs/tutorials/security/cluster-level-pss/](https://kubernetes.io/docs/tutorials/security/cluster-level-pss/)

## Pod Admission Policies

In the CKA section of this workshop, you enabled the ValidatingAdmissionPolicy plugin on the kube-apiserver. This plugin allows you to enforce policies on pods before they are created. The policies are defined using CEL expressions and be a powerful tool to enforce your organization's security policies.

### Using Validating Admission Policies

Run the following command to ValidatingAdmissionPolicy that ensures deployment creation and updates have less than or equal to 5 replicas.

```bash
kubectl apply -f - <<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "demo-policy.example.com"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
EOF
```

Next, bind the policy to a namespace and set the validation action to "Warn".

```bash
kubectl apply -f - <<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "demo-policy-binding.example.com"
spec:
  policyName: "demo-policy.example.com"
  validationActions: ["Warn"]
  matchResources:
    namespaceSelector:
      matchLabels:
        "kubernetes.io/metadata.name": "default"
EOF
```

Test the policy by creating a deployment with more than 5 replicas.

```bash
kubectl create deploy mynginx --image=nginx --replicas=10
```

You will see the deployment was created but since the validation action was set to "Warn", the deployment will be created with a warning.

```text
Warning: Validation failed for ValidatingAdmissionPolicy 'demo-policy.example.com' with binding 'demo-policy-binding.example.com': failed expression: object.spec.replicas <= 5
deployment.apps/mynginx created
```

Update the policy binding to deny the deployment creation.

```bash
kubectl apply -f - <<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "demo-policy-binding.example.com"
spec:
  policyName: "demo-policy.example.com"
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        "kubernetes.io/metadata.name": "default"
EOF
```

Delete the deployment.

```bash
kubectl delete deploy mynginx
```

Try creating the deployment again.

```bash
kubectl create deploy mynginx --image=nginx --replicas=10
```

Now you will see the deployment was denied with a failure message.

```text
error: failed to create deployment: deployments.apps "mynginx" is forbidden: ValidatingAdmissionPolicy 'demo-policy.example.com' with binding 'demo-policy-binding.example.com' denied request: failed expression: object.spec.replicas <= 5
```

This was a fairly simple example of ValidatingAdmissionPolicy. To see more complex examples, checkout the link below.

**Reference:** [https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/](https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/)

## Runtime Security

### Sandboxing with gVisor

By using different runtimes, you can isolate workloads and improve security. gVisor is a user-space kernel that provides an additional layer of security between the container and the host kernel. It is designed to be lightweight and efficient, and it provides a secure environment for running untrusted workloads.

!!! note
    In the Kubernetes setup, you installed gVisor as a runtime class and configured containerd to use it. 

SSH into a worker node, switch to the root user, then run the following command on any of the worker nodes to verify that gVisor is installed as a runtime class.

```bash
runsc --version
```

Run the following command to confirm containerd is configured to use the runsc runtime.

```bash
cat /etc/containerd/config.toml | grep runsc
```

SSH into the control node and run the following command to create a runtime class for gVisor.

```bash
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF
```

Use the runtime class in a pod spec.

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-gvisor
spec:
  runtimeClassName: gvisor   # Use the runtime class here
  containers:
  - name: nginx
    image: nginx
EOF
```

If you run the following command in the container, you should see that gVisor is started as the runtime.

```bash
kubectl exec -it nginx-gvisor -- dmesg
```

**Reference:** [https://kubernetes.io/docs/concepts/containers/runtime-class/](https://kubernetes.io/docs/concepts/containers/runtime-class/)


```bash
kubectl apply -f https://gist.githubusercontent.com/pauldotyu/0daeda395c7896e12cbd0f34c69d38e4/raw/6fcba9c08807d233069927a4804fc5f795e2f9aa/insecure-rabbitmq.yaml
```

## Control plane component security

### Secrets in etcd

etcd is a distributed key-value store that is used by Kubernetes to store configuration data. It is used to store cluster state and configuration data, and it is a critical component of a Kubernetes cluster.

Secrets are stored in etcd and are encrypted at rest. However, it is important to ensure that etcd is properly secured to prevent unauthorized access to secrets.

Run the following command to create a secret in the default namespace.

```bash
kubectl create secret generic mydatabasecreds --from-literal=username=admin --from-literal=password=mysupersecretpassword
```

Run the following command to get the secret data out of etcd.

```bash
etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt get /registry/secrets/default/mydatabasecreds 
```

You will see that the secret data is there in plain text. So even if a user does not have access to the Kubernetes API, or has restricted permissions to view secrets, they can still access the secrets directly from etcd.

### Encrypting secrets in etcd

SSH into the control node and switch to the root user.

Run the following command to create a new encryption key.

```bash
BASE64_ENCODED_SECRET=$(head -c 32 /dev/urandom | base64)
```

Create a new EncryptionConfiguration file with the encryption key.

```bash
mkdir /etc/kubernetes/enc
cat <<EOF > /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${BASE64_ENCODED_SECRET}
      - identity: {} # remove this line to prevent plain text retrieval
EOF
```

Make a copy of the existing kube-apiserver manifest.

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml.bak
```

Using Vim, edit the kube-apiserver manifest to include the encryption configuration to the list of kube-apiserver flags.

```bash
- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

Next add the appropriate volume and volume mount so the encryption configuration file can be mounted in the kube-apiserver pod.

```bash
    volumeMounts:                         # this is in the container spec
    ...
    - mountPath: /etc/kubernetes/enc      # add these 3 lines to the bottom of the list
      name: enc                           
      readOnly: true                      
    ...
  volumes:                                # this is in the pod spec
  ...
  - hostPath:                             # add these 4 lines to the bottom of the list
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
    name: enc
```

Save and exit the kube-apiserver manifest then wait for the kube-apiserver pod to restart.

```bash
watch crictl ps
```

After you see the kube-apiserver pod has been restarted, run the following command to update the secret which will now be encrypted.

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Now if you try to get the secret data out of etcd, you will see that it is encrypted.

```bash
etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt get /registry/secrets/default/mydatabasecreds | hexdump -C
```

**Reference:** [https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

## Scanning images for vulnerabilities

Scanning images for vulnerabilities is an important part of securing your Kubernetes cluster. There are a number of tools that can be used to scan images for vulnerabilities. One such tool is Trivy.

### Using Trivy

Trivy is a simple and comprehensive vulnerability scanner for containers. It can be used to scan container images for vulnerabilities and generate reports.

Trivy is not typically installed on Kubernetes nodes. It is more commonly used in CI/CD pipelines to scan images before they are deployed to a cluster.

Let's install Trivy on the control node and scan the `nginx:latest` image for vulnerabilities.

SSH into the control node and run the following command to install Trivy.

```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

**Reference:** [https://trivy.dev/v0.57/getting-started/installation/](https://trivy.dev/v0.57/getting-started/installation/)


Run the following command to scan the `nginx:latest` image for vulnerabilities with a severity of HIGH or CRITICAL.

```bash
trivy image --severity HIGH,CRITICAL nginx:latest
```

You will see that the image has a few vulnerabilities and some of which are marked `will_not_fix`. You can filter out the `will_not_fix` vulnerabilities by running the following command.

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:latest
```

Trivy has the ability to scan more than just Docker images. It can also scan configuration files and other artifacts like file systems, infrastructure as code files like Terraform, sbom files, and more.

Be sure to check out the Trivy documentation linked below for more information.

**Reference:** [https://trivy.dev/v0.57/tutorials/overview/](https://trivy.dev/v0.57/tutorials/overview/)