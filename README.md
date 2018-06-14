# kubeproject

Provisioning a cluster from scratch with Docker and Kubernetes

This project aims to try each different way to provision a Kubernetes cluster from scratch. There are several ways to do it, more or less long. I'll start with the longest and tedious way, configuring every aspect of the cluster, and then reducing the difficulty using designed tools to provision a Kubernetes cluster.
I'm going to split different techniques explanation across different docs.

* Introduction:
  * [Docker](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/docker.md)
  * [Kubernetes](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/kubernetes.md)

* The provisioning and building of the cluster is explained in decreasing difficulty across these files:

  * [kubernetes-the-hard-way](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/kubernetes-the-hard-way.md)
  * [kubernetes-the-kubeadm-way](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/kubernetes-the-kubeadm-way.md)
  * [kubernetes-the-kubespray-way](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/kubernetes-the-kubespray-way.md)

* After the provisioning I'm going to create a sample WordPress website running in Docker containers:

  * [Helm](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/helm.md)
  * [Deploying WordPress with Helm](https://github.com/xn3cr0nx/kubeproject/blob/master/docs/deploying-wordpress-with-helm.md)