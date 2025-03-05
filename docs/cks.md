## Overview

The last exam in the path to become a Kubestronaut is the CKS exam. This exam is probably the most difficult exam to clear. You will be tested on your ability to not only secure Kubernetes clusters and workloads but also on your ability to respond to security incidents and investigate security breaches. This includes things like setting up network policies, securing the etcd database, using external security tools, and more.

Read through the [Certified Kubernetes Security Specialist (CKS)](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist-cks-2/) page for Domain and Competency details and information on how to register for the exam.

Some of the hands on activities that you should be comfortable with are:

- Using kernel hardening tools like AppArmor and seccomp
- Using Falco to monitor runtime security
- Using CIS benchmarks to secure a cluster
- Performing static analysis of workloads and container images
- Implementing end-to-end traffic encryption with Cilium
- Using network policies to restrict traffic to applications
- Using Pod security standards to secure workloads 
- Securing the kube-apiserver

## Scanning images for vulnerabilities

Scanning images for vulnerabilities is an important part of securing your Kubernetes cluster. There are a number of tools that can be used to scan images for vulnerabilities. One such tool is Trivy.

### Using Trivy

[Trivy](https://trivy.dev/latest/docs/target/container_image/) is a simple and comprehensive vulnerability scanner. It can be used to scan container images for vulnerabilities and generate reports. It is important to ensure that images with vulnerabilities are resolved as soon as possible to prevent security breaches.

Trivy isn't typically installed on Kubernetes nodes. It is more commonly used in CI/CD pipelines to scan images before they are deployed to a cluster. But we installed it on the control node for the purpose of this workshop.

Let's use Trivy on the control node and scan the `nginx:latest` image for vulnerabilities.

SSH into the control node then run the following command to scan the `nginx:latest` image for vulnerabilities with a severity of HIGH or CRITICAL.

```bash
trivy image --severity HIGH,CRITICAL nginx:latest
```

!!! warning
    This can take a few minutes to complete as trivy needs to download the vulnerability database.

After the scan has completed, you'll see that the image has a few vulnerabilities and some of which are marked `will_not_fix`. You can filter out the `will_not_fix` vulnerabilities by running the following command.

```bash
trivy image --severity HIGH,CRITICAL --ignore-status will_not_fix nginx:latest
```

There are no vulnerabilities in the `nginx:latest` image that are marked as `will_not_fix`. But in the event you do see vulnerabilities that are fixable, you can refer to the **Fixed Version** column to see what version of the package the vulnerability was fixed in. Typically, remediations can be applied by using newer versions of images if you are using third party images or by updating the base images in your Dockerfiles if you are building your own images.

Trivy has the ability to scan more than just Docker images. It can also scan configuration files and other artifacts like file systems, infrastructure as code files like Terraform, sbom files, and more.

**Reference:** [https://trivy.dev/v0.57/tutorials/overview/](https://trivy.dev/v0.57/tutorials/overview/)

## Network Policies

I can't stress this enough, being proficient with network policies is critical to passing all the Kubernetes exams. Network policies are used to control the flow of traffic to and from pods. They are a powerful tool to secure your workloads and are a key component of securing a Kubernetes cluster. We did a bit of network policy exercises in the previous sections, but it is worth noting that in addition to standard Kubernetes network policies, Cilium also provides [network policy](https://docs.cilium.io/en/latest/security/policy/index.html) that can be used to secure network traffic in a Kubernetes cluster. 

Cilium is a powerful networking and security tool that can be used to secure your Kubernetes cluster. It provides a number of features including network policies, encryption, and more.

### Using CiliumNetworkPolicy

Create the nginx1 and nginx2 pods we'll use for testing.

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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx2
spec:
  selector:
    app: nginx2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

Run the following commands to create Cilium network policies that:

1. Block all incoming traffic to nginx2
2. Allow nginx2 to send traffic to nginx1

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

Test the network policy by trying to access the nginx2 pod from the nginx1 pod and confirm that the connection times out.

```bash
kubectl exec -it nginx1 -- curl nginx2 --connect-timeout 1
```

Now, try to access the nginx1 pod from the nginx2 pod and confirm that the connection is successful.

```bash
kubectl exec -it nginx2 -- curl nginx1
```

CiliumNetworkPolicy may not appear on the CKS exam, but the concepts in using it is similar to standard Kubernetes network policies. CiliumNetworkPolicy is a powerful tool that gives you additional control over network traffic in your cluster like applying DNS-based or layer 7 policies.

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

Since we initially installed Cilium without encryption, we can enable it by running the following command.

```bash
cilium config set enable-wireguard true
```

!!! info
    To enable encryption in Cilium with Wireguard on install, you can run the following command.

    ```bash
    cilium install --version 1.17.1 \
    --set encryption.enabled=true \
    --set encryption.type=wireguard
    ```

Wait a few minutes for the Pods to restart then run the following command to confirm that encryption is enabled.

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

To verify traffic is being sent through the encrypted interface, switch to run as the root user then run the following commands to install and run tcpdump on the cilium_wg0 interface.

```bash
apt-get update
apt-get -y install tcpdump
tcpdump -n -i cilium_wg0
```

!!! warning
    Make sure you switched to the root user using the `sudo -i` command before running the commands above.

You should see encrypted traffic being sent through the cilium_wg0 interface.

```text
istening on cilium_wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
01:34:45.400904 IP 192.168.120.136.50597 > 192.168.120.135.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.1.1.4240 > 10.0.0.105.43484: Flags [.], ack 1345254179, win 504, options [nop,nop,TS val 3516968635 ecr 3092550065], length 0
01:34:45.401149 IP 192.168.120.135.36724 > 192.168.120.136.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.105.43484 > 10.0.1.1.4240: Flags [.], ack 1, win 510, options [nop,nop,TS val 3092565428 ecr 3516952919], length 0
01:34:49.335788 IP 192.168.120.135.37207 > 192.168.120.136.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.105 > 10.0.1.1: ICMP echo request, id 52988, seq 0, length 32
01:34:49.336123 IP 192.168.120.135.36724 > 192.168.120.136.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.105.43484 > 10.0.1.1.4240: Flags [P.], seq 1:100, ack 1, win 510, options [nop,nop,TS val 3092569363 ecr 3516952919], length 99
01:34:49.337391 IP 192.168.120.136.38801 > 192.168.120.135.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.1.1 > 10.0.0.105: ICMP echo reply, id 52988, seq 0, length 32
01:34:49.337494 IP 192.168.120.136.50597 > 192.168.120.135.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.1.1.4240 > 10.0.0.105.43484: Flags [P.], seq 1:76, ack 100, win 504, options [nop,nop,TS val 3516972573 ecr 3092569363], length 75
01:34:49.337547 IP 192.168.120.135.36724 > 192.168.120.136.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.105.43484 > 10.0.1.1.4240: Flags [.], ack 76, win 510, options [nop,nop,TS val 3092569365 ecr 3516972573], length 0
01:34:50.284495 IP 192.168.120.137.41721 > 192.168.120.135.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.2.230.41078 > 10.0.0.29.4240: Flags [.], ack 3241835173, win 510, options [nop,nop,TS val 2574638965 ecr 3776558958], length 0
01:34:50.284658 IP 192.168.120.135.56975 > 192.168.120.137.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.0.29.4240 > 10.0.2.230.41078: Flags [.], ack 1, win 504, options [nop,nop,TS val 3776588986 ecr 2574623994], length 0
01:34:50.673277 IP 192.168.120.135.56975 > 192.168.120.137.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.0.29.4240 > 10.0.2.230.41078: Flags [.], ack 1, win 504, options [nop,nop,TS val 3776589375 ecr 2574623994], length 0
01:34:50.674070 IP 192.168.120.137.41721 > 192.168.120.135.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.2.230.41078 > 10.0.0.29.4240: Flags [.], ack 1, win 510, options [nop,nop,TS val 2574639355 ecr 3776588986], length 0
^C
11 packets captured
11 packets received by filter
0 packets dropped by kernel
```

When done, press `Ctrl+C` to stop tcpdump then exit the root user by running the `exit` command.

**Reference:** [https://docs.cilium.io/en/stable/security/network/encryption/](https://docs.cilium.io/en/stable/security/network/encryption/)

## Pod Security Standards

Pod security standards are a set of best practices that you can use to secure your workloads. These standards are enforced by the kube-apiserver and can be used to ensure that your workloads are secure.

Enabling Pod Security Standards is done by labeling a namespace with a label. This label can be set to various values to enforce different levels of security. The values are:

- **Privileged** - No restrictions, provides the most flexibility but is the least secure
- **Baseline** - Prevents known privilege escalations with minimal restrictions
- **Restricted** - Heavily restricted Pod configuration following hardening best practices

**Reference:** [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/) 

### Enabling Pod Security Standards

Run the following command to label the default namespace with the `pod-security.kubernetes.io/enforce` label set to `restricted`.

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

You will see the pod wasn't created because the pod can allow privilege escalation.

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

In the CKA section of this workshop, you [enabled the ValidatingAdmissionPolicy plugin on the kube-apiserver](https://pauldotyu.github.io/kubestronaut/cka/#enable-validatingadmissionpolicy). This plugin allows you to enforce policies on pods before they are created. The policies are defined using [CEL expressions](https://kubernetes.io/docs/reference/using-api/cel/) and be a powerful tool to enforce your organization's security policies.

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

Update the policy binding to "Deny" the deployment creation.

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

Now you'll see the deployment was denied with a failure message.

```text
error: failed to create deployment: deployments.apps "mynginx" is forbidden: ValidatingAdmissionPolicy 'demo-policy.example.com' with binding 'demo-policy-binding.example.com' denied request: failed expression: object.spec.replicas <= 5
```

This was a fairly simple example of ValidatingAdmissionPolicy. To see more complex examples, checkout the link below.

**Reference:** [https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/](https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/)

## Runtime security

Runtime security focuses on protecting your applications while they're running. This includes isolating workloads from each other and from the host system, detecting anomalous behavior, and responding to security events in real-time. Below we explore gVisor as one runtime security approach.

### Sandboxing with gVisor

By using different runtimes, you can isolate workloads and improve security. gVisor is a user-space kernel that provides an additional layer of security between the container and the host kernel. It is designed to be lightweight and efficient, and it provides a secure environment for running untrusted workloads.

!!! note
    In the [Kubernetes setup](https://pauldotyu.github.io/kubestronaut/kubernetes/#install-runsc) section, gVisor was installed as a runtime class and configured containerd to use it. 

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
  nodeName: worker-1
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
sudo etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt get /registry/secrets/default/mydatabasecreds 
```

You will see that the secret data is there in plain text. So even if a user doesn't have access to the Kubernetes API, or has restricted permissions to view secrets, if an adversary has access to a control plane node, they can still access the secrets directly from etcd.

### Encrypting secrets in etcd

Encrypting data at rest in etcd is a good practice to prevent unauthorized access.

To begin, SSH into the control node and switch to the root user.

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

Using Vim, edit the kube-apiserver manifest to include `- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml` to the list of kube-apiserver command flags.

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Next scroll down towards the bottom of the manifest and add the appropriate volume and volume mount so the encryption configuration file can be mounted in the kube-apiserver pod.

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

After you see the kube-apiserver pod has been restarted, press `Ctrl+C` to exit the watch then run the following command to update the secret which will now be encrypted.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Now if you try to get the secret data out of etcd, you'll see that it is encrypted.

```bash
etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt get /registry/secrets/default/mydatabasecreds | hexdump -C
```

**Reference:** [https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

## Additional Resources

- [CKS Exam Curriculum](https://github.com/cncf/curriculum/blob/master/CKS_Curriculum%20v1.32.pdf)
- [CKS Certification Learning Path](https://kodekloud.com/learning-path/cks)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [Community curated CKS resources](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist)