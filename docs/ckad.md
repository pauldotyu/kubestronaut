## Overview

Once you get past the KCNA and KCSA exams, you should have your sights set on the CKAD exam. This was once considered an entry-level exam but due to its intense hands-on nature, it's better classified as intermediate. Despite this, it makes an excellent starting point for those looking to dive deeper into Kubernetes.

Read through the [Certified Kubernetes Application Developer (CKAD)](
https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/) page for Domain and Competency details and information on how to register for the exam.

Some of the hands on activities that you should be comfortable with are:

- Deploying applications and choosing the right workload resource type for the job (i.e., Deployment, StatefulSet, DaemonSet, Job, CronJob)
- Configuring applications using ConfigMaps and Secrets
- Creating and managing Persistent Volumes and Persistent Volume Claims
- Using ServiceAccounts to give applications the right permissions
- Using network policies to restrict traffic to applications
- Using Ingress resources to expose applications to the outside world

## Deploying Applications

A [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) is the smallest deployable unit in Kubernetes. It is a logical host for one or more containers which runs your application.

You would not typically create Pods directly, but rather use a workload resource to manage the pods for you. There are several different types of workload resources in Kubernetes, each with its own use case and knowing when to use each is important. The most common types of workload resources are:

- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/): A deployment is a declarative way to manage a set of replicas of a pod. It is the most common way to deploy applications in Kubernetes.
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/): A replica set is a workload resource that is used to ensure that a specified number of pod replicas are running at any given time. It is used to manage the scaling of applications.
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/): A stateful set is a workload resource that is used to manage stateful applications. It is used for applications that require stable, unique network identifiers and stable storage.
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/): A daemon set is a workload resource that ensures that a copy of a pod is running on all or some nodes in the cluster. It is used for applications that need to run on every node in the cluster, such as logging or monitoring agents.
- [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/): A job is a workload resource that is used to run a batch job. It is used for applications that need to run to completion, such as data processing jobs.
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/): A cron job is a workload resource that is used to run a batch job on a schedule. It is used for applications that need to run on a schedule, such as backups or report generation.

The resources that you request are reconciled by various [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) in the Kubernetes control plane. For example, when you create a Deployment, the Deployment controller will create a ReplicaSet and the ReplicaSet controller will create the Pods. When you submit a resource through the Kubernetes API, the desired state is stored in etcd and controllers are responsible for ensuring that the actual state matches the desired state. This is known as the reconciliation loop.

!!! info
    Kubernetes also supports custom controllers through its [extension patterns](https://kubernetes.io/docs/concepts/extend-kubernetes/#extension-patterns) which allow you to extend the functionality of Kubernetes.

### Use kubectl to generate YAML

In the KCNA section of this workshop, you used kubectl imperative commands to deploy applications. That is a good way to quickly get started but it isn't the best way to deploy applications in production. In production, you should be using declarative configuration files in the form of YAML manifests to deploy applications. This allows you to version control application deployments and makes it easier to manage changes over time.

Getting started with YAML might seem daunting at first, but it can be made easy if you leverage a feature of kubectl and have it generate the YAML for you. 

Run the following command to create a namespace.

```bash
kubectl create namespace n654
```

Let's deploy the application that you packaged up in the KCNA section of this workshop.

Run the following command to create a deployment, but this time use the `--dry-run=client` and `-o yaml` flags to generate the YAML for you and redirect the output to a file called **myapp2.yaml**.

```bash
kubectl create deployment myapp2 \
--namespace n654 \
--image=<PASTE_IN_YOUR_CONTAINER_IMAGE_NAME> \
--replicas=3 \
--port=3000 \
--dry-run=client \
-o yaml > myapp2.yaml
```

!!! warning
    This assumes you have a container image that you built in the KCNA section of this workshop.

View the **myapp2.yaml** file.

```bash
cat myapp2.yaml
```

In the output, you'll see that the entire Deployment manifest has been generated for you. If needed, you can edit the file to make any changes that you want and then apply it.

Run the following command to apply the manifest.

```bash
kubectl apply --namespace n654 -f myapp2.yaml
```

The application should now be deployed. Run the following command to get the Pods.

```bash
kubectl get pods --namespace n654
```

Why stop there? We can add the Service and Ingress resources to the same file too.

In YAML manifests, you can have multiple resources in the same file as long as they are separated by three dashes. So let's add that then append the service YAML to the file.

```bash
echo "---" >> myapp2.yaml
```

Then run the following command to generate the Service YAML and append it to the file.

```bash
kubectl expose deployment myapp2 \
--namespace n654 \
--port=80 \
--target-port=3000 \
--dry-run=client \
-o yaml >> myapp2.yaml
```

Do the same for creating an Ingress for the application to expose it externally using a load balancer.

```bash
echo "---" >> myapp2.yaml
```

Then run the following command to generate the ingress YAML and append it to the file.

```bash
kubectl create ingress myapp2 \
--namespace n654 \
--class=nginx \
--rule="myapp2.example.com/*=myapp2:80" \
--dry-run=client \
-o yaml >> myapp2.yaml
```

Now, view the **myapp2.yaml** file and notice how it has all three resources in it.

```bash
cat myapp2.yaml
```

Finally, apply the manifest.

```bash
kubectl apply --namespace n654 -f myapp2.yaml
```

!!! tip
    You can use the `--dry-run=client` and `-o yaml` flags to generate the YAML for any resource type. Throughout the exam, you should leverage this technique as much as possible to save time. It is easier to use kubectl to generate most of the YAML for you than it is to write it from scratch.

If all went well, you should be able to access the application from your host machine using the following URL.

```bash
curl http://control -H "Host: myapp2.example.com"
```

### Use kubectl for reference

kubectl is also a great tool for reference. You can use it to get information about resources in the cluster. 

To see all the available resources in the cluster, run the following command.

```bash
kubectl api-resources
```

The output will show you all the resources that are available in the cluster. You will see the short name, full name, and the namespaced status of each resource. The short name in particular is useful for quickly referencing resources in kubectl commands. Remember, time is of the essence so you save a few seconds by using the short name. For example, you can use `po` instead of `pods`. Combine this with command aliases and you can save even more time.

You can run something like this to get information about an Ingress resource.

```bash
k get ing -n n654
```

You can also use the `explain` command to get detailed information about a resource spec. For example, if you want to know more about the Ingress resource, you can run the following command.

```bash
kubectl explain ingress
```

To dig deeper into the spec, you can run the following command.

```bash
kubectl explain ingress.spec
```

You can also use the `--recursive` flag to get information about nested fields, but it won't give you the full details of each field.

```bash
kubectl explain ingress.spec --recursive
```

The `--help` flag is also incredibly useful. You can use it to get information about the flags that are available for a command. It will even give you examples of how to use the command.

Run this command and see what you get.

```bash
kubectl create configmap --help
```

!!! tip
    kubectl is your friend. Use it to get information you need to get a task done. It is faster than looking up the documentation.

## Configuring Applications

All applications need to be configured in some way. In Kubernetes, there are two main ways to configure applications: using [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) and [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

ConfigMaps are used to store non-sensitive configuration data in key-value pairs. They can be used to store configuration files, command-line arguments, environment variables, and more. ConfigMaps can be mounted as volumes or exposed as environment variables in pods.

Secrets are used to store sensitive information, such as passwords, OAuth tokens, and SSH keys. Secrets stored in etcd and are base64 encoded --not encrypted; which is a key consideration when using them. Anyone with access to the etcd database can read the secrets. Therefore, it is important to use[ encryption at rest for etcd data](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).

Applications may also need to be configured to use persistent storage. In Kubernetes, persistent storage is managed using [Persistent Volumes (PV)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and [Persistent Volume Claims (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). A PV is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. A PVC is a request for storage by a user.

### Using PVs and PVCs

PVs and PVCs are used to manage persistent storage usage in Kubernetes. To use persistent storage, you need to create a PV to represent a chunk of storage and a PVC to request the storage --an allocation. There are several different types of PVs, including hostPath, NFS, iSCSI, and cloud provider-specific PVs. 

!!! note
    Since we are working with a local Kubernetes cluster and all we have are local disks, we will use the hostPath type of PV. The hostPath type of PV uses a file or directory on the host to store the data. This is useful for testing and development purposes but isn't recommended for production use.

Run the following command to create a PV on the host's /tmp directory. We don't have much storage space to work with so we'll only carve out 20Mi of storage.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
 name: my-pv
spec:
 capacity:
  storage: 20Mi
 accessModes:
   - ReadWriteOnce
 hostPath:
  path: "/tmp"
EOF
```

!!! note
    The PV is a cluster resource that represents a piece of storage in the cluster so a namespace isn't needed.

Run the following command to view the PV. You should see that the PV is available. This is because the PV hasn't been claimed by a PVC yet.

```bash
kubectl get pv -n n239
```

Now create a PVC that requests the 20Mi of storage.

```bash
kubectl create ns n239
kubectl apply -n n239 -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Mi
EOF
```

If you run the command to view the PV again, you'll see that the PV is bound to the PVC.

```bash
kubectl get pv -n n239
```

Additionally, if following command to view the PVC, you'll see that the PVC is linked to the PV.

```bash
kubectl get pvc -n n239
```

PVCs are then used in workload resources and mounted as volumes in the Pod spec. Run the following command to create a Pod that uses the PVC.

```bash
kubectl apply -n n239 -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-pod
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:                # Define the volume mounts for the container
      - name: data               # Name of the volume
        mountPath: /data         # Path to mount the volume in the container
  volumes:                       # Define the volumes for the Pod
    - name: data                 # Name of the volume
      persistentVolumeClaim:     # Source type of the volume
        claimName: my-pvc        # Name of the PVC
EOF
```

If you run the following command, you should be able to view the contents of the mounted volume in the Pod.

```bash
kubectl exec -it my-pod -n n239 -- ls /data
```

Since we used the host's /tmp directory for the PV, the Pod now has access to the host's /tmp directory and all the files within it.

!!! info
    This is a security risk and should be avoided in production. Another concern of using hostPath is that the data is limited to the host that the Pod was scheduled on. If the Pod is rescheduled to another host, the data will not be available.

### Using ConfigMaps

There are a few ways to use ConfigMaps in Kubernetes. The most common way is to use them as environment variables in a pod. This allows you to pass configuration data to your application without hardcoding it into the application itself.

In this example, you'll create a ConfigMap and use it to configure an application.

Run the following command to create a temporary file to store plugin configuration for RabbitMQ.

```bash
cat << EOF | tee /tmp/rabbitmq_enabled_plugins
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_amqp1_0].
EOF
```

Create a namespace and a ConfigMap from the RabbitMQ plugins file.

```bash
kubectl create ns n754
kubectl create configmap rabbitmq-enabled-plugins -n n754 --from-file=/tmp/rabbitmq_enabled_plugins
```

Next, run the following command to use the ConfigMap as a file mounted in the RabbitMQ Pod to enable the plugins.

```bash
kubectl apply -n n754 -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: rabbitmq
        image: rabbitmq:3.10-management-alpine
        ports:
        - containerPort: 5672
          name: rabbitmq-amqp
        - containerPort: 15672
          name: rabbitmq-http
        env:
        - name: RABBITMQ_DEFAULT_USER
          value: "username"
        - name: RABBITMQ_DEFAULT_PASS
          value: "password"
        resources:
          requests:
            cpu: 10m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        volumeMounts:                                 # Define the volume mounts for the container
        - name: rabbitmq-enabled-plugins              # Name of the volume
          mountPath: /etc/rabbitmq/enabled_plugins    # Path to mount the volume in the container
          subPath: enabled_plugins                    # Mount as a file in the container
      volumes:                                        # Define the volume for the Pod
      - name: rabbitmq-enabled-plugins                # Name of the volume
        configMap:                                    # Source type of the volume
          name: rabbitmq-enabled-plugins              # Name of the ConfigMap
          items:                                      # Specify the items to mount
          - key: rabbitmq_enabled_plugins             # Name of the key in the ConfigMap
            path: enabled_plugins                     # Name of the file in the Pod
EOF
```

Based on the comments in the YAML, you can see that we're mounting the ConfigMap as a file in the container. The `subPath` field is used to specify the name of the file in the container. This allows us to mount a single key from the ConfigMap as a file in the container.

If you run the following command, you should be able to view the contents of the plugins file in the RabbitMQ Pod.

```bash
kubectl exec -it rabbitmq-0 -n n754 -- cat /etc/rabbitmq/enabled_plugins | grep rabbitmq_amqp1_0
```

The other ways to use ConfigMaps is to mount them as environment variables, so be sure to check out ths [guide](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) on how to do that.

**Reference:** [https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/)

### Using Secrets

Secrets are used to store "sensitive" information. I put that in quotes because secrets aren't really secret. They are base64 encoded and anyone with access to the etcd database can read them. In terms of usage, they are similar to ConfigMaps. You can use them as environment variables or mount them as volumes.

If you noticed in the previous example, the username and password for RabbitMQ were hardcoded in the YAML. This isn't a good practice. Instead, you should use a Secret to store the username and password.

Run the following command to create a temporary file to store the RabbitMQ username and password.

```bash
kubectl create secret generic rabbitmq-secret -n n754 --from-literal=RABBITMQ_DEFAULT_USER=username --from-literal=RABBITMQ_DEFAULT_PASSWORD=password
```

Run the following command to use the Secret as an environment variable in the RabbitMQ Pod to set the admin user and password, and expose the RabbitMQ management interface on port 15672.

```bash
kubectl apply -n n754 -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: rabbitmq
        image: rabbitmq:3.10-management-alpine
        ports:
        - containerPort: 5672
          name: rabbitmq-amqp
        - containerPort: 15672
          name: rabbitmq-http
        envFrom:                       # Use envFrom to load all keys from the secret as environment variables
        - secretRef:                   # Reference the secret
            name: rabbitmq-secret      # Name of the secret
        resources:
          requests:
            cpu: 10m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
EOF
```

If you run the following command, you'll see that the username and password are now being set as environment variables in the RabbitMQ Pod.

```bash
kubectl describe pod rabbitmq-0 -n n754 | grep "Environment Variables from" -A 1
```

## Exposing Applications

Understanding the different types of services is also important. There are four types of services in Kubernetes: [ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip), [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport), [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer), and [ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname). 

The most common type of service is ClusterIP, which exposes the service on a cluster-internal IP. This means that the service is only accessible from within the cluster. NodePort exposes the service on each node's IP at a static port. This means that the service is accessible from outside the cluster by requesting the node IP on the node port. LoadBalancer exposes the service externally using a cloud provider's load balancer, which ultimately gives you a public or private IP address. ExternalName maps the service to the contents of the externalName field (e.g., foo.bar.example.com), by returning a CNAME record with its value.

It is also important to remember that services uses the selector to find the pods that it should route traffic to. This is done using labels. Labels are key-value pairs that are attached to resources in Kubernetes.

### Using Services

Run the following command to expose the RabbitMQ deployment that you just created.

```bash
kubectl apply -n n754 -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  ports:
    - name: rabbitmq-amqp
      port: 5672
      targetPort: 5672
    - name: rabbitmq-http
      port: 15672
      targetPort: 15672
  selector:
    app: rabbitmq      # The selector is used to find the pods that the service should route traffic to
EOF
```

With the service created, you can access the nginx application using the service name. Run the following command to get the service IP.

```bash
kubectl run mycurl -n n754 --image=curlimages/curl -it --rm --restart=Never -- curl rabbitmq:15672
```

### Using Ingress

The default service type is ClusterIP, which means that the service is only accessible from within the cluster. We demonstrated this with the curl command above with the curl pod running in the same cluster, you were able to access the RabbitMQ service using the service name. But services especially HTTP-based services are often meant to be accessed from outside the cluster. To do this, you can use an Ingress resource.

Think of an Ingress as a reverse proxy that routes traffic to the appropriate service based on the request URL. It is most commonly used to route HTTP and HTTPS traffic, but it can also be used to route TCP and UDP traffic.

Run the following command to create an Ingress resource for the RabbitMQ service.

```bash
kubectl apply -n n754 -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rabbitmq
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /      # Rewrite the URL to remove the path prefix
spec:
  ingressClassName: nginx
  rules:
  - host: rabbitmq.example.com
    http:
      paths:
      - backend:
          service:
            name: rabbitmq
            port:
              number: 15672
        path: /manage                                  # Request path to the RabbitMQ management interface
        pathType: Prefix
      - backend:
          service:
            name: rabbitmq
            port:
              number: 5672
        path: /amqp                                    # Request path to the RabbitMQ AMQP interface
        pathType: Prefix
EOF
```

!!! note
    Notice that the Ingress resource is using path-based routing to route traffic to the appropriate service based on the request URL. The path /manage is used to route traffic to the RabbitMQ management interface, while the path /amqp is used to route traffic to the RabbitMQ AMQP interface. But these paths mean nothing to the actual RabbitMQ service so you're using the `nginx.ingress.kubernetes.io/rewrite-target` annotation to rewrite the URL to remove the path prefix before it is sent to the RabbitMQ service.

Now you can access the RabbitMQ management interface from your host machine. Run the following command to access the RabbitMQ management interface.

```bash
curl http://control/manage -H "Host: rabbitmq.example.com"
```

!!! warning
    Make sure you run the command from your host machine and not from the control node.

## Securing network traffic

Network policies are extremely important to understand for all of the hands-on exams. In the previous section, we were introduced to NetworkPolicy but it is worth revisiting it here as it is a major part of all the Kubernetes exams and tends to be a stumbling block for many test takers.

In Kubernetes, there are two important behaviors to understand regarding network policies:

1. By default, with no NetworkPolicies applied, all pods can communicate with all other pods
2. Once you apply any NetworkPolicy with a podSelector to a namespace, the selected pods become isolated and only accept traffic explicitly allowed by NetworkPolicies.

### Deny all ingress traffic

Run the following command to create a namespace and a deployment.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

!!! warning
    This assumes you deployed the AKS Store Demo microservices app in the [previous section](https://pauldotyu.github.io/kubestronaut/kcna/#microservices) section of this workshop. If you've not already, please go back and deploy it before you proceed.

The deny-all policy you created earlier (with `podSelector: {}`) selects all pods in the namespace and defines no ingress rules, effectively blocking all incoming traffic to every pod.

Let's test the network policies to confirm all ingress traffic is blocked.

Run the following command to get the pod name of the store-front application.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=store-front -o jsonpath='{.items[0].metadata.name}')
```

Connection from store-front to product-service should timeout.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://product-service:3002/health --timeout=1
```

Connection from store-front to order-service should also timeout.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://order-service:3000/health --timeout=1
```

Finally, get the pod name of the order-service and confirm the connection from order-service to RabbitMQ also times out.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=order-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://rabbitmq:15672/ --timeout=1
```

### Configure specific traffic patterns

All traffic is blocked at the moment. Let's open up ingress traffic to the product-service from the store-front application. This is a common use case for network policies. You want to restrict access to a service to only allow traffic from specific applications.

Run the following command to allow ingress on the product-service only from the store-front application.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-storefront-to-product-service
spec:
  podSelector:
    matchLabels:
      app: product-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: store-front
  policyTypes:
  - Ingress
EOF
```

Similarly, the order-service should also only be accessible from the store-front application. So let's create a network policy for that as well.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-storefront-to-order-service
spec:
  podSelector:
    matchLabels:
      app: order-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: store-front
  policyTypes:
  - Ingress
EOF
```

Finally, the RabbitMQ service should only be accessible from the order-service application. So let's create a network policy for that as well.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-service-to-rabbitmq
spec:
  podSelector:
    matchLabels:
      app: rabbitmq
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: order-service
  policyTypes:
  - Ingress
EOF
```

Run the following command to get the pod name of the store-front application.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=store-front -o jsonpath='{.items[0].metadata.name}')
```

Connection from store-front to product-service should now be successful.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://product-service:3002/health --timeout=1
```

Connection from store-front to order-service should now be successful as well.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://order-service:3000/health --timeout=1
```

Finally, get the pod name of the order-service and confirm the connection from order-service to RabbitMQ should now be successful.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=order-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://rabbitmq:15672/ --timeout=1
```

Network policies are a powerful tool for securing your Kubernetes cluster. It is important to understand how they work and how to use them effectively. It is also important to efficiently test your network policies to ensure that they are working as expected during the exam.

For more NetworkPolicy practice, check out the [NetworkPolicy Editor by Isovalent](https://editor.networkpolicy.io/) for an interactive experience.

## Additional Resources

There is a lot more to cover for the CKAD exam. Here are some additional resources to help you prepare:

- [CKAD Exam Curriculum](https://github.com/cncf/curriculum/blob/master/CKAD_Curriculum_v1.32.pdf)
- [CKAD Certification Learning Path](https://kodekloud.com/learning-path/ckad)
- [CKAD - free materials](https://www.reddit.com/r/kubernetes/comments/r4q1ec/ckad_free_materials/)