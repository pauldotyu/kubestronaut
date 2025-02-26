The KCNA exam is considered to be the entry level exam. It is a multiple choice exam that tests your overall understanding of fundamental Kubernetes and Cloud Native architecture concepts.

Take a look at the [Kubernetes and Cloud Native Associate (KCNA)](https://training.linuxfoundation.org/certification/kubernetes-cloud-native-associate/) page for Domain and Competency details.

Some of the topics that you should be familiar with include:

- Container fundamentals
- Kubernetes fundamentals
- Cloud Native architecture
- Cloud Native observability tools
- Cloud Native application delivery tools

Let's practice with some exercises to help familiarize yourself with the concepts.

## Container fundamentals

A container is a application packaging method that allows you to run an application and its dependencies in a isolated environment. Containers are lightweight and portable, making them ideal for deploying applications in a cloud native environment.

A container image is defined within a Dockerfile. The Dockerfile contains instructions on how to build the image. The image is then used to create a container.

!!! warning
    This section will require you to work in your local machine (not in the virtual machine) and have Docker and Node.js installed on your local machine.
    
    If you don't have Docker installed, follow the instructions on the [Docker installation page](https://docs.docker.com/get-docker/) to install Docker.
    
    If you don't have Node.js installed, follow the instructions on the [Node.js installation page](https://nodejs.org/en/download/) to install Node.js.

### Create a simple Node.js application

Let's create a simple Node.js application and package it in a container. Open a terminal and follow the steps below:

Create a new directory for the application:

```bash
mkdir myapp
cd myapp
```

Initialize a new Node.js application.

```bash
npm init
```

Install the `express` package.

```bash
npm install express
```

Create a new directory called `src`. This is where application source code will live.

```bash
mkdir src
cd src
```

Create a file called `server.js` and add the following code.

```javascript
const express = require("express");
const app = express();
const server = app.listen(3000, () => {
  console.log(`Express running → PORT ${server.address().port}`);
});
app.get("/", (req, res) => {
  res.send("Hello World!");
});
```

Return to the `myapp` directory and run the application.

```bash
cd ..
node src/server.js
```

Open a browser and navigate to http://localhost:3000. You should see "Hello World!" displayed.

Press `Ctrl+C` to stop the application.

### Package the application in a container

Now that we have a simple Node.js application, let's package it in a container. We will use Docker to build the container image.

Create a new file called `Dockerfile` in the `myapp` directory and add the following code.

```Dockerfile
FROM node:22
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
EXPOSE 3000
COPY . .
CMD ["node", "src/server.js"]
```

Build the container image.

```bash
docker build -t myapp:latest .
```

Run the application in a container.

```bash
docker run -d -p 3000:3000 myapp:latest
```

Open a browser and navigate to http://localhost:3000. You should see "Hello World!" displayed.

Run the following command to list the running containers.

```bash
docker ps
```
Run the following command to stop the container.

```bash
docker stop <CONTAINER_ID> # replace <CONTAINER_ID> with the actual container ID
```

!!! tip
    You don't need to pass in the entire container ID. You can just pass in the first few characters of the container ID.

### Multi-stage builds

The Dockerfile above is fairly simple, but it can be optimized using multi-stage builds. Multi-stage builds allow you to use multiple `FROM` statements in a single Dockerfile. This can help reduce the size of the final image and is often used as a best practice in terms of security and performance.

**Reference:** [docker-multi-stage-builds]{:target="_blank"}

Run the following command to list the container images.

```bash
docker images
```

Note the size of the `myapp:latest` image is a little over 1.6GB. It is quite large because it contains all the dependencies and build artifacts.

Open the `Dockerfile` and replace the contents with the following code.

```Dockerfile
# Build stage
FROM node:22 AS build
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .

# Production stage
FROM node:22-slim
WORKDIR /usr/src/app
COPY --from=build /usr/src/app .
EXPOSE 3000
CMD ["node", "src/server.js"]
```

This Dockerfile has two stages: a build stage and a production stage. The build stage is used to install dependencies and build the application, while the production stage is used to run the application using a "slim" version of the Node.js image. This often results in a smaller final image size and reduces the attack surface.

Build the container image again.

```bash
docker build -t myapp:latest .
```

Run the following command to list the container images.

```bash
docker images
```

Note that the size of the `myapp:latest` image is now a little over 340MB. This is a significant reduction in size. Not only that, but the final image only contains the files needed to run the application, which is a good security practice.

If you build and run the container image using the same commands as before, you should see the same "Hello World!" message displayed.

### Multi-architecture builds

The promise of containers is that they can run anywhere. But in reality, this is not always the case. Different operating system architectures require different container images. For example, a container image built for AMD64 architecture will not run on ARM64 architecture.

Good news is that Docker gives you the ability to build container images for different architectures. The docker CLI includes a plugin feature called buildx that allows you to build multi-arch images. This allows you to run the same container image on different platforms without modification.

You can create a new buildx builder by running the following command.

```bash
docker buildx create --name mybuilder --driver docker-container --platform linux/amd64,linux/arm64,linux/arm64/v8 --bootstrap --use --node mybuilder
```

!!! note
    You can also configure Builders in your Docker Desktop settings.

**Reference:** [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/)

Now you can use the new buildx builder to create a multi-arch image.

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

### Container registries

Container images can be stored in container registries, such as Docker Hub, Azure Container Registry, or GitHub Container Registry. This allows you to share images with others and deploy them to different environments.

There are many guides available on how to push and pull images from different container registries. But for this exercise, we'll use [ttl.sh](https://ttl.sh/) which is an ephemeral and anonymous container registry offered by friends at [Replicated](https://www.replicated.com/).

Container images pushed to ttl.sh have a limited lifetime and are automatically deleted after a specified period. So, you need to to give the image a unique repository name and give it a lifetime in the form of a tag. The tag is a string that represents the lifetime of the image. For example, `4h` means the image will be deleted after 4 hours.

Run the following commands to create a unique repository name and tag.

```bash
IMG_REPO=$(uuidgen | tr '[:upper:]' '[:lower:]')
IMG_VERSION=4h
```

Build and push the container image to ttl.sh.

```bash
docker buildx build --platform linux/amd64,linux/arm64 --push -t ttl.sh/${IMG_REPO}:${IMG_VERSION} .
```

Now if you run the following command, you should see the image being pulled from ttl.sh.

```bash
docker pull ttl.sh/${IMG_REPO}:${IMG_VERSION}
```

Run the following command to print the container image you pushed to ttl.sh and copy the output of the command to a text file for future reference.

```bash
echo ttl.sh/${IMG_REPO}:${IMG_VERSION}
```

!!! note
    Remember this container image will be deleted after 4 hours.

## Kubernetes fundamentals

Now that you have a basic understanding of containers, let's move on to Kubernetes. Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

### Kubernetes architecture and components

Quickly review the [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/) documentation to understand the key components of a Kubernetes cluster. It is a great way to get started with Kubernetes and will help you understand the basic concepts.

The cluster is made up of a control plane and one or more nodes. The control plane is responsible for managing the cluster and the nodes are responsible for running the applications. You created a control plane node and joined worker nodes to the cluster using the kubeadm tool in the previous section. 

Be sure to read through the [Control plane components](https://kubernetes.io/docs/concepts/architecture/#control-plane-components) and [Node components](https://kubernetes.io/docs/concepts/architecture/#node-components) sections to understand the roles of each component.k

Trying to learn Kubernetes fundamentals in a short period of time is a bit of an impossible task. But here in this workshop, we can cover some of the basics by deploying our Node.js application to a Kubernetes cluster.

!!! note
    You will use kubectl to perform imperative commands to deploy the application. This is good to quickly perform a task, but using declarative configuration files in the form of YAML manifests is the best way to deploy applications in production. You will visit this more in the next few sections.

### Create a Namespace

Namespaces are a way to logically separate resources in a cluster. They are intended for use in environments with many users spread across multiple teams, or projects. Create a Namespace called `myapp` to isolate the resources for our Node.js application by running the following command.

```bash
kubectl create namespace n345
```

### Run a container in a Pod

Containers don't run on their own in Kubernetes. They run inside a Pod. A Pod is the smallest deployable unit in Kubernetes and represents a single instance of a running process in your cluster. To run a Pod, the easiest way is to use kubectl, which is the Kubernetes command-line tool that allows you to interact with your cluster and use an imperative command to create a Pod.

SSH back into the control node and run the following command to create a Pod that runs the Node.js application.

```bash
kubectl run myapp \
--namespace n345 \
--image=<PASTE_IN_YOUR_CONTAINER_IMAGE_NAME> \
--port=3000
```

!!! warning
    Make sure you are connected to your virtual machine using SSH.

Run the following command to list the Pods in your cluster.

```bash
kubectl get pods --namespace n345
```

### Expose the Pod with a Service

The Pod is running, but it is not accessible from outside the cluster. To make it accessible, you need to expose the Pod as a Service. A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy by which to access them.

Run the following command to expose the Pod as a Service.

```bash
kubectl expose pod myapp \
--namespace n345 \
--type=NodePort \
--port=80 \
--target-port=3000
```

!!! info
    Notice that we are using the `--type=NodePort` flag to expose the Pod as a NodePort Service. This means that the Service will be exposed on a port on each Node in the cluster. There are other types of Services, such as ClusterIP and LoadBalancer, and you should read the [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/) to learn more about them.

Run the following command to list the Services in your cluster.

```bash
kubectl get services --namespace n345
```

You will see output similar to the following:

```text
NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
myapp   NodePort   10.107.207.155   <none>        80:31519/TCP   4s
```

The `myapp` Service is now exposed on port `31519` on your virtual machine. You can access the Node.js application by navigating to [http://control:31519](http://control:31519) in your browser.

!!! note
    Your Service port might be different than `31519`, so make sure to check the output of the `kubectl get services` command and adjust the URL accordingly.

### Scale the application with Deployments and ReplicaSets

Running a single Pod is fine for testing, but in a production environment, you would want to run multiple instances of your application for redundancy and load balancing. You can scale the application by increasing the number of replicas in the Deployment.

Let's create a Deployment that manages the Pods and scale the application to three replicas.

Run the following command to delete the Pod and Service.

```bash
kubectl delete pod myapp --namespace n345 --wait=false
kubectl delete service myapp --namespace n345 --wait=false
```

Run the following command to create a Deployment that manages the Pods.

```bash
kubectl create deployment myapp \
--namespace n345 \
--image=<PASTE_IN_YOUR_CONTAINER_IMAGE_NAME> \
--replicas=3 \
--port=3000
```

Run the following command to list the Deployments in your cluster.

```bash
kubectl get deployments --namespace n345
```

Now if you check the Pods, you should see three Pods running.

```bash
kubectl get pods --namespace n345
```

Did you notice that the Pods have different names? Kubernetes automatically generates a unique name for each replica. A replica is a copy of the Pod that is managed by a ReplicaSet. So from a mental model perspective, you can think of a Deployment as a higher-level abstraction that manages a ReplicaSet, which in turn manages the Pods. You can see this relationship by running the following command. 

```bash
kubectl describe replicasets --namespace n345
```

You should see that the ReplicaSet is "Controlled By" the `myapp` Deployment.

### Exposing web applications with Ingress

In the section above, we exposed the Node.js application Pod using a Service. You can also expose the Deployment as a Service, but when it comes to exposing web applications, it is common to use Ingress.

Ingress is an API object that manages external access to services in a cluster. It provides HTTP and HTTPS routing to Services based on hostnames and paths. Ingress is often configured to provide load balancing, SSL termination, and name-based virtual hosting.

To use Ingress, you need to have an Ingress Controller running in your cluster. The Ingress Controller is a daemon that watches the Kubernetes API server for Ingress resources and updates the configuration of a load balancer in the cluster. You will also need to have a Service in place that the Ingress Controller can route traffic to.

Run the following command to create an Ingress resource that routes traffic to the `myapp` Service.

```bash
kubectl expose deployment myapp \
--namespace n345 \
--port=80 \
--target-port=3000
```

Run the following command to create an Ingress resource that routes traffic to the `myapp` Service.

```bash
kubectl create ingress myapp \
--namespace n345 \
--class=nginx \
--rule="myapp.example.com/*=myapp:80"
```

Now you can access the Node.js application from your local machine using the following curl command.

```bash
curl http://control -H 'Host: myapp.example.com'
```

!!! warning
    Run this command from your local machine, not the virtual machine.

## Cloud Native architecture

Cloud Native architecture is a design philosophy that leverages cloud computing principles to build and run scalable applications in modern environments. It's a set of practices that help organizations deliver applications faster and more reliably.

### Twelve-factor app

It will be good to understand some of the key concepts of the [Twelve-factor app](https://12factor.net/) and how they apply to Cloud Native architecture.

### Microservices

One important element of the Twelve-factor app architecture is treating applications as a set of loosely coupled services. This is also known as the microservices architecture. Microservices are small, independent services that work together to form a larger application. They communicate with each other over a network, typically using HTTP or messaging protocols. With microservices, you can break down complex applications into smaller, more manageable components. This allows you to develop, deploy, and scale each component independently.

This architectural approach is demonstrated in the [AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo?tab=readme-ov-file#architecture) app. 

Run the following command to deploy this sample application to your Kubernetes cluster.

```bash
kubectl create ns pets
kubectl apply -n pets -f https://gist.githubusercontent.com/pauldotyu/bfa405733af19bc80c357d6aab5847c7/raw/3f0dd08fcc7a0fb1662369efd0ab47c8b4d0d533/aks-store-demo.yaml
```

Run the following command to list the Deployments in the `pets` namespace.

```bash
kubectl get deployments -n pets
```

The application is made up of multiple Deployments, each managing a set of Pods. There is a store-front web app written using NodeJS that is main user interface for a customer. Products are retrieved using a REST API provided by the product-service which is written in Rust. Orders are processed via the order-service which is also a REST API which is an ExpressJS app which sends order messages to RabbitMQ which is a commonly used message broker. So each of these components are written in different languages and are deployed as separate microservices but connected together via APIs to form a complete application. 

Access the application by navigating to the Ingress IP address in your browser. Run the following command to get the URL based on the IP address of Ingress in the `pets` namespace.

```bash
echo "http://$(kubectl get ingress -n pets store-front -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

## Cloud Native observability tools

Observability is the ability to understand the internal state of a system by examining the outputs of its components. In a Cloud Native environment, where applications are distributed and run across multiple containers, it is important to have tools that provide visibility into the system. Some of the key observability tools include:

- [Prometheus](https://prometheus.io/docs/introduction/overview/): A time-series database that collects metrics from monitored targets commonly used for monitoring Kubernetes clusters.
- [Grafana](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/): A visualization tool that works with Prometheus to create visualizations and dashboards.
- [Jaeger](https://www.jaegertracing.io/docs/2.3/): A distributed tracing system that helps you monitor and troubleshoot transactions in complex systems.
- [Fluentd](https://docs.fluentd.org/): A data collector that unifies data collection and consumption for better use and understanding of data.

## Cloud Native application delivery tools

Application delivery is the process of getting software into the hands of users in a safe and reliable way. In a Cloud Native environment, where applications are built and deployed using containers, it is important to have tools that automate the delivery process. Some of the key application delivery tools include:

- [Helm](https://helm.sh/): A package manager for Kubernetes that helps you define, install, and upgrade applications.
- [Kustomize](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/): A tool that lets you customize Kubernetes resources through a kustomization file.
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/): A declarative, GitOps continuous delivery tool for Kubernetes.
- [Flux](https://fluxcd.io/flux/concepts/): A tool that automatically ensures that the state of your Kubernetes cluster matches the configuration you've supplied in Git.

## Additional resources

There's plenty more to learn about to pass the KCNA exam, but this should give you a good starting point. Go checkout the [CNCF's KCNA curriculum](https://github.com/cncf/curriculum/tree/master/kcna) for more resources.

<!-- URLs -->
[docker-multi-stage-builds]: https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/