# kubeproject
Provisioning a cluster from scratch with Docker and Kubernetes

This project aims to try each different way to provision a Kubernetes cluster from scratch. There are several ways to do it, more or less long. I'll start with the longest and tedious way, configuring every aspect of the cluster, and then reducing the difficulty using designed tools to provision a Kubernetes cluster.
I'm going to split different techniques docs across different docs. 
- The provisioning and building of the cluster is explained in decreasing difficulty across these files:
	- [kubernetes-the-hard-way.m](docs/kubernetes-the-hard-way.m)
	- kubernetes-the-kubeadm-way.m
	- kubernetes-the-kubespray-way.m
	- kubernetes-the-kargo-way.m
	- kubernetes-the-easy-way.m

- After the provisioning I'm going to create a sample application running in Docker containers:
	- containerizing-the-app.m

- Deploying of the application and configuration of logging and monitoring addos via different ways:
	- deploying-from-scratch.m
	- deploying-with-helm-charts.m 