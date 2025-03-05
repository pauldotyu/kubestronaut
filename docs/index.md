---
hide:
  - feedback
---

## Overview 

So you want to be a [Kubestronaut](https://www.cncf.io/training/kubestronaut/)? You've landed in the right place!

Passing five Kubernetes exams sounds challenging but intrinsically rewarding. Earning a Kubernetes certification is empowering, especially if you're new to the technology. It demonstrates your ability to deploy, manage, and secure Kubernetes environments, which are essential skills for any cloud native engineer. 

The path to Kubestronaut is definitely an arduous journey that requires dedication, hands-on practice, and a structured approach. This workshop is designed to kick start your journey, by helping you set up a local Kubernetes lab environment to practice and prepare for the exams. 

I won't cover every single certification exams topic in depth, but I will provide some hands-on exercises and tips I think will be helpful based on my own experience from preparing for and passing the exams.

## Learning objectives

By the end of this workshop, you'll be able to:

- Set up a local multi-node Kubernetes lab environment using [VMware Desktop Hypervisor](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion) 
- Install and configure [Ubuntu](https://ubuntu.com/download/server) server with proper networking for Kubernetes nodes
- Build a Kubernetes cluster from scratch using [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) with [containerd](https://containerd.io/) runtime
- Configure essential cluster components including [Cilium CNI](https://cilium.io/use-cases/cni/), [MetalLB](https://metallb.io/), and [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/)
- Practice hands-on exercises commonly found on Kubernetes exams (CKAD, CKA, CKS)
- Master effective exam techniques, including proper documentation navigation and time management strategies
- Gain practical troubleshooting experience using tools like vim, sed, systemctl, and journalctl

!!! note 
    You have other options for running a local Kubernetes lab environment, such as using [Minikube](https://minikube.sigs.k8s.io/docs/), [Kind](https://kind.sigs.k8s.io/), or [K3s](https://k3s.io/). But I chose to bootstrap a cluster with kubeadm on Ubuntu servers because it provides an environment that closely resembles a production cluster.

## Pre-requisites

Most of the exercises in this workshop will be completed within a virtual machine. But you should also have the following tools also installed on your local machine to complete some of the exercises in this workshop:

- POSIX-compliant shell (bash, zsh, etc.)
- [Docker Desktop](https://www.docker.com/get-started/)
- [Node.js](https://nodejs.org/en/download)
- [Trivy](https://trivy.dev/latest/getting-started/installation)

## Additional resources

There are many resources available to help you prepare for the Kubernetes certification exams. Some of the most useful resources include:

- [CNCF Kubernetes Certification Programs](https://www.cncf.io/training/certification/) - Official certification information
- [Kubernetes Documentation](https://kubernetes.io/docs/) - The comprehensive reference you'll use during exams
- [Killer.sh](https://killer.sh/) - Simulator for CKA, CKAD, and CKS exams (included with exam purchase)
- [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/) - Tutorials for beginners
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Deep-dive guide to manual cluster setup

This workshop started as a series of blog posts on my blog site [https://paulyu.dev](https://paulyu.dev) so you can find the original content there as well as other cloud native content.

Let's begin your journey to becoming a Kubestronaut 🚀
