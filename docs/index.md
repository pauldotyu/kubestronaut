---
hide:
  - feedback
---

So you want to become a Kubestronaut? You've landed in the right place!

The path to Kubernetes certification is challenging but intrinsically rewarding. Passing a Kubernetes certification exam is empowering, especially if you're new to the technology. It demonstrates your ability to deploy, manage, and secure Kubernetes environments, which are essential skills for any cloud-native engineer. 

The path to Kubestronaut is definitely an arduous journey that requires dedication, hands-on practice, and a structured approach. This workshop is designed to guide you through that journey, by helping you set up a local Kubernetes lab environment to practice and prepare for the exams. 

I will not cover every single topic in the certification exams in depth, but I will provide some hands-on exercises and tips I think will be helpful based on my own experience from preparing for and passing the exams.

This workshop started as a series of blog posts on my blog site [https://paulyu.dev](https://paulyu.dev) so you can find the original content there as well as other cloud native content.

## Learning objectives

My main goal of this workshop is to get you setup with local Kubernetes lab environment and provide you with some hands-on exercises to help you prepare for the exams. Here are some of the learning objectives:

- Set up a complete local Kubernetes development environment using VMware virtualization
- Install and configure Ubuntu server for use as a Kubernetes node
- Install and configure a multi-node Kubernetes cluster from scratch using kubeadm with containerd and Cilium
- Develop practical skills through hands-on exercises to master some tricky Kubernetes tasks required for each certification
- Understand the differences between the five main Kubernetes certifications and how to prepare for each
- Learn effective exam strategies, including time management and documentation navigation

!!! note 
    You have other options for running a local Kubernetes lab environment, such as using Minikube, Kind, or K3s. But I chose to use VMware virtualization because it provides a more realistic environment that closely resembles a production cluster.

## Pre-requisites

Most of the exercises in this workshop will be completed within a virtual machine. But you should also have the following tools also installed on your local machine:

- POSIX-compliant shell (bash, zsh, etc.)
- [Docker Desktop](https://www.docker.com/get-started/)
- [Node.js](https://nodejs.org/en/download)
- [Trivy](https://trivy.dev/latest/getting-started/installation)

## Additional resources

There are many resources available to help you prepare for the Kubernetes certification exams. Some of the most useful resources include:

- [CNCF Kubernetes Certification Programs](https://www.cncf.io/training/certification/) - Official certification information
- [Kubernetes Documentation](https://kubernetes.io/docs/) - The comprehensive reference you'll use during exams
- [Killer.sh](https://killer.sh/) - Simulator for CKA, CKAD, and CKS exams (included with exam purchase)
- [Certified Kubernetes Security Specialist Study Guide](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist) - Community-maintained CKS resources
- [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/) - Tutorials for beginners
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Deep-dive guide to manual cluster setup

Let's begin your journey to becoming a Kubestronaut!
