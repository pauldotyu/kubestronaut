Once you get passed the KCNA and KCSA exams, you should have your sights on the CKAD exam. This was once considered to be the entry level exam but due to its intense hands-on nature, I would consider it to be the intermediate level exam. This should be the starting point for anyone looking to get into the world of Kubernetes and being a Kubernetes developer; that is, being able to deploy, scale, and maintain applications in Kubernetes.

Take a look at the [Certified Kubernetes Application Developer (CKAD)](
https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/) page for Domain and Competency details.

Some of the hands on activities that you should be comfortable with are:

- Deploying applications and choosing the right workload resource type for the job (i.e., Deployment, StatefulSet, DaemonSet, Job, CronJob)
- Configuring applications using ConfigMaps and Secrets
- Creating and managing Persistent Volumes and Persistent Volume Claims
- Using ServiceAccounts to give applications the right permissions
- Using network policies to restrict traffic to applications
- Using Ingress resources to expose applications to the outside world

## Deploying Applications

Applications in Kubernetes are deployed using workload resources. There are several different types of workload resources in Kubernetes, each with its own use case and knowing when to use each is important. The most common types of workload resources are:

- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/): A deployment is a declarative way to manage a set of replicas of a pod. It is the most common way to deploy applications in Kubernetes.
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/): A replica set is a workload resource that is used to ensure that a specified number of pod replicas are running at any given time. It is used to manage the scaling of applications.
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/): A stateful set is a workload resource that is used to manage stateful applications. It is used for applications that require stable, unique network identifiers and stable storage.
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/): A daemon set is a workload resource that ensures that a copy of a pod is running on all or some nodes in the cluster. It is used for applications that need to run on every node in the cluster, such as logging or monitoring agents.
- [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/): A job is a workload resource that is used to run a batch job. It is used for applications that need to run to completion, such as data processing jobs.
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/): A cron job is a workload resource that is used to run a batch job on a schedule. It is used for applications that need to run on a schedule, such as backups or report generation.

All the workload resources are used to manage pods. A [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) is the smallest deployable unit in Kubernetes. It is a logical host for one or more containers. 

!!! note
    You would typically not create pods directly, but rather use a workload resource to manage the pods for you.

The resources that you request are reconciled by various controllers in the Kubernetes control plane. For example, when you create a deployment, the deployment controller will create a replica set and the replica set controller will create the pods. When you submit a resource to the Kubernetes API, the desired state is stored in etcd and controllers are responsible for ensuring that the actual state matches the desired state. This is known as the reconciliation loop.

### Quickly generate YAML

In the KCNA section of this workshop, we used kubectl imperative commands to deploy applications. That is a good way to quickly get started but it is not the best way to deploy applications in production. In production, you should be using declarative configuration files in the form of YAML manifests to deploy applications. This allows you to version control your application deployments and makes it easier to manage changes over time.

Getting started with YAML might seem daunting at first, but it can be made easy if you leverage a feature of kubectl and have it generate the YAML for you. 

Run the following command to create a namespace.

```bash
kubectl create namespace n654
```

Let's deploy the application that you packaged up in the KCNA section of this workshop.

Run the following command to create a deployment, but this time use the `--dry-run=client` and `-o yaml` flags to generate the YAML for you and redirect the output to a file called **myapp.yaml**.

```bash
kubectl create deployment myapp \
--namespace n654 \
--image=<PASTE_IN_YOUR_CONTAINER_IMAGE_NAME> \
--replicas=3 \
--port=3000 \
--dry-run=client \
-o yaml > myapp.yaml
```

View the **myapp.yaml** file.

```bash
cat myapp.yaml
```

In the output, you'll see that the entire deployment has been generated for you. If needed, you can edit the file to make any changes that you want and then apply it.

Why stop there? We can add the service and ingress resources to the same file.

In YAML manifests, you can have multiple resources in the same file as long as they are separated by three dashes. So let's add that then append the service YAML to the file.

```bash
echo "---" >> myapp.yaml
```

Then run the following command to generate the service YAML and append it to the file.

```bash
kubectl expose deployment myapp \
--namespace n654 \
--port=80 \
--target-port=3000 \
--dry-run=client \
-o yaml >> myapp.yaml
```

And do the same for creating an ingress for the application to expose it externally using a load balancer.

```bash
echo "---" >> myapp.yaml
```

Then run the following command to generate the ingress YAML and append it to the file.

```bash
kubectl create ingress myapp \
--namespace n654 \
--class=nginx \
--rule="myapp.example.com/*=myapp:80" \
--dry-run=client \
-o yaml >> myapp.yaml
```

Now, view the **myapp.yaml** file and notice how it has all three resources in it.

```bash
cat myapp.yaml
```

Finally, apply the manifest.

```bash
kubectl apply --namespace n654 -f myapp.yaml
```

So you can simply use the `--dry-run=client` and `-o yaml` flags to generate the YAML for any resource type. Throughout the exam, you should leverage this technique as much as possible to save time. It is easier to use kubectl to generate most of the YAML for you than it is to write it from scratch.

If all went well, you should be able to access the application from your host machine using the following URL.

```bash
curl http://control -H "Host: myapp.example.com"
```

## Configuring Applications

All applications need to be configured in some way. In Kubernetes, there are two main ways to configure applications: using ConfigMaps and Secrets.

ConfigMaps are used to store non-sensitive configuration data in key-value pairs. They can be used to store configuration files, command-line arguments, environment variables, and more. ConfigMaps can be mounted as volumes or exposed as environment variables in pods.

Secrets are used to store sensitive information, such as passwords, OAuth tokens, and SSH keys. Secrets are stored in etcd and are base64 encoded; which is a key consideration when using them. Anyone with access to the etcd database can read the secrets. Therefore, it is important to use encryption at rest for etcd data.

Applications may also need to be configured to use persistent storage. In Kubernetes, persistent storage is managed using Persistent Volumes (PV) and Persistent Volume Claims (PVC). A PV is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. A PVC is a request for storage by a user.

### Using ConfigMaps

There are a few ways to use ConfigMaps in Kubernetes. The most common way is to use them as environment variables in a pod. This allows you to pass configuration data to your application without hardcoding it into the application itself.

In this example, we will create a ConfigMap and use it to configure an application.

Run the following command to create a temporary file to store plugins for RabbitMQ.

```bash
cat << EOF | tee /tmp/rabbitmq_enabled_plugins
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_amqp1_0].
EOF
```

Run the following command to create a namespace and a ConfigMap from the RabbitMQ plugins file.

```bash
kubectl create ns n754
kubectl create configmap rabbitmq-enabled-plugins -n n754 --from-file=/tmp/rabbitmq_enabled_plugins
```

Run the following command to use the ConfigMap as a file mounted in the RabbitMQ Pod to enable the plugins.

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

Based on the comments in the YAML, you can see that we are mounting the ConfigMap as a file in the container. The `subPath` field is used to specify the name of the file in the container. This allows us to mount a single key from the ConfigMap as a file in the container.

If you run the following command, you should be able to view the contents of the plugins file in the RabbitMQ Pod.

```bash
kubectl exec -it rabbitmq-0 -n n754 -- cat /etc/rabbitmq/enabled_plugins | grep rabbitmq_amqp1_0
```

The other ways to use ConfigMaps is to mount them as environment variables, so be sure to check out the [Kubernetes tutorial](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/) on how to do that.

### Using Secrets

Secrets are used to store "sensitive" information. I put that in quotes because secrets are not really secret. They are base64 encoded and anyone with access to the etcd database can read them. In terms of usage, they are similar to ConfigMaps. You can use them as environment variables or mount them as volumes.

If you noticed in the previous example, the username and password for RabbitMQ were hardcoded in the YAML. This is not a good practice. Instead, you should use a Secret to store the username and password.

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

If you run the following command, you will see that the username and password are now being set as environment variables in the RabbitMQ Pod.

```bash
kubectl describe pod rabbitmq-0 -n n754 | grep "Environment Variables from" -A 1
```

### Using PVs and PVCs

Persistent Volumes (PV) and Persistent Volume Claims (PVC) are used to manage persistent storage in Kubernetes. To use persistent storage, you need to create a PV to represent the storage and a PVC to request the storage. There are several different types of PVs, 

Run the following command to create a PV.

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
    The PV is a cluster resource that represents a piece of storage in the cluster so a namespace is not needed.

Run the following command to create a PVC that requests the 20Mi of storage.

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

Run the following command to view the PV. You should see that the PV is bound to the PVC.

```bash
kubectl get pv -n n239
```

Run the following command to view the PVC. You should see that the PVC is attached to the PV.

```bash
kubectl get pvc -n n239
```

Run the following command to create a Pod that uses the PVC.

```bash
kubectl apply -n n239 -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynginx
  labels:
    app: mynginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mynginx
  template:
    metadata:
      labels:
        app: mynginx
    spec:
      containers:
        - name: container
          image: nginx:latest
          volumeMounts:                            # Define the volume mounts for the container
            - name: data                           # Name of the volume
              mountPath: /usr/share/nginx/html     # Path to mount the volume in the container
      volumes:                                     # Define the volumes for the Pod
        - name: data                               # Name of the volume
          persistentVolumeClaim:                   # Source type of the volume
            claimName: my-pvc                      # Name of the PVC
EOF
```

If you run the following command, you should be able to view the contents of the mounted volume in the nginx Pod.

```bash
kubectl describe deploy mynginx -n n239 | grep "Mounts" -A 6
```

## Exposing Applications

Understanding the different types of services is also important. There are four types of services in Kubernetes: ClusterIP, NodePort, LoadBalancer, and ExternalName. The most common type of service is ClusterIP, which exposes the service on a cluster-internal IP. This means that the service is only accessible from within the cluster. NodePort exposes the service on each node's IP at a static port. This means that the service is accessible from outside the cluster by requesting the node IP on the node port. LoadBalancer exposes the service externally using a cloud provider's load balancer, which ultimately gives you a public or private IP address. ExternalName maps the service to the contents of the externalName field (e.g., foo.bar.example.com), by returning a CNAME record with its value.

It is also important to remember that services uses the selector to find the pods that it should route traffic to. This is done using labels. Labels are key-value pairs that are attached to resources in Kubernetes.

### Using Services

Run the following command to expose the RabbitMQ deployment that you created in the previous section.

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

The default service type is ClusterIP, which means that the service is only accessible from within the cluster. We demonstrated this with the curl command above with the curl pod running in the same cluster, we were able to access the RabbitMQ service using the service name. But services especially HTTP-based services are often meant to be accessed from outside the cluster. To do this, you can use an Ingress resource.

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
    Notice that the Ingress resource is using path-based routing to route traffic to the appropriate service based on the request URL. The path /manage is used to route traffic to the RabbitMQ management interface, while the path /amqp is used to route traffic to the RabbitMQ AMQP interface. But these paths mean nothing to the actual RabbitMQ service so we are using the `nginx.ingress.kubernetes.io/rewrite-target` annotation to rewrite the URL to remove the path prefix before it is sent to the RabbitMQ service.

Now you can access the RabbitMQ management interface from your host machine. Run the following command to access the RabbitMQ management interface.

```bash
curl http://control/manage -H "Host: rabbitmq.example.com"
```

## Securing network traffic

Network policies are extremely important to understand for all of the hands-on exams. In the previous section, we created some basic network policies to restrict traffic to applications. In this section, we will create some more advanced network policies to restrict traffic to applications.

### Deny all ingress traffic

By default, all ingress traffic is allowed in a Kubernetes cluster. This means that any pod can communicate with any other pod in the cluster. This is not a good security practice. To restrict ingress traffic, you can use network policies.

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

### Allow ingress traffic from a specific pod

NetworkPolicy rules are additive. This means that if you create a network policy that allows ingress traffic from a specific pod, it will not override the default allow-all policy. So you need to delete the deny-all policy first.

```bash
kubectl delete -n pets networkpolicy deny-all
```

Now if you browse to the store-front application again, you will see that it is not able to access the product-service.

Now let's limit the ingress traffic to the product-service to only allow traffic from the store-front application. This is a common use case for network policies. You want to restrict access to a service to only allow traffic from specific applications.

Run the following application to allow ingress on the product-service only from the store-front application.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-storefront
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

Same goes for the order-service, it should only be accessible from the store-front application. So let's create a network policy for that as well.

```bash
kubectl apply -n pets -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-storefront
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
  name: allow-order-service
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

Now let's test the network policies. Run the following command to get the pod name of the store-front application.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=store-front -o jsonpath='{.items[0].metadata.name}')
```

Run the following command to exec into the store-front application and curl the product-service.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://product-service:3002
```

Run the following command to exec into the store-front application and curl the order-service.

```bash
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://order-service:3000/health
```

Run the following command to confirm the product-service is not able to be reached from any other application.

```bash
POD_NAME=$(kubectl get pod --namespace pets -l app=order-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME --namespace pets -- wget -qO- http://product-service:3002
```


