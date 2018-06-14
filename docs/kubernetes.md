# Kubernetes

## Docker Limits

Docker, by itself, is very good at managing single containers. When you start using more and more containers and containerized apps, broken down into hundreds of pieces, management and orchestration can get very difficult. Eventually, you need to take a step back and group containers to deliver services, networking, security, telemetry, etc. across all of your containers. That's where Kubernetes comes in.

## What is k8s

K8s is an open-source container-orchestration system for automating deployment, scaling and management of containerized applications. It eliminates many of the manual processes involved in deploying and scaling containerized applications. In other words, you can cluster together groups of hosts running Linux containers, and Kubernetes helps you easily and efficiently manage those clusters. A Kubernets cluster is made up by a lot of components, these are the most important:

- Master: The machine that controls Kubernetes nodes. This is where all task assignments originate.
- Node: These machines perform the requested, assigned tasks. The Kubernetes master controls them.
- Pod: A group of one or more containers deployed to a single node. All containers in a pod share an IP address, IPC, hostname, and other resources. Pods abstract network and storage away from the underlying container. This lets you move containers around the cluster more easily.
- Replication controller: This controls how many identical copies of a pod should be running somewhere on the cluster.
- Service: This decouples work definitions from the pods. Kubernetes service proxies automatically get service requests to the right pod—no matter where it moves to in the cluster or even if it’s been replaced.
- Kubelet: This service runs on nodes and reads the container manifests and ensures the defined containers are started and running.
- kubectl: This is the command line configuration tool for Kubernetes.

> I'm going to explore all pieces k8s is made up in the Kubernetes The Hard Way section.

![Kubernetes Architecture](https://res.cloudinary.com/dukp6c7f7/image/upload/f_auto,fl_lossy,q_auto/s3-ghost/2016/06/o7leok.png)

## Why do you need Kubernetes?

Real production apps span multiple containers. Those containers must be deployed across multiple server hosts. Kubernetes gives you the orchestration and management capabilities required to deploy containers, at scale, for these workloads. Kubernetes orchestration allows you to build application services that span multiple containers, schedule those containers across a cluster, scale those containers, and manage the health of those containers over time.

All Rights Reserved © 2018 Red Hat, Inc©.  
All rights reserved © 2018 The Linux Foundation®.
