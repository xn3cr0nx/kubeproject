# Helm

Helm is a package manager for Kubernetes (we can think about it as well as apt for Linux or npm for NodeJS).  
Helm helps you to manage Kubernetes applications (define, install, upgrade)

In order to create a Kubernetes application you just need to install che Helm stable chart (for example here I'm going to install a WordPress website from its stable chart). Anyway you could create your own chart e deploy your custom application.

Helm is made up by two parts, a client (helm) and a server (tiller).

* Tiller: runs inside your Kubernetes cluster and manages releases (e.g. installations) of charts.
* Charts: Helm packages that contain at least a description (Chart.yaml) and one or more templates (k8s manifest files)

Tiller and Helm are written in Go language. Tiller communicates with the kubernetes api

## Installing

First of all you need Kubernetes installed and a local configured copy of kubectl. Helm's actions are based on the context of your cluster, this means you need to set the context as I explained in the last part of kubernets the kubespray way.

I'm just going to explain the faster way to install it, but you could download binary release and install it too.  
Anyway with this command you will download the script and launch it that automatically will install Helm client.
`curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash`

Now you just need to install the Tiller server. In orderd to do this you just need to launch `helm init`. I'm going to insert this command in the next and last chapter too.

You should see a command like
> $HELM_HOME has been configured at /Users/varMyUser/.helm.  
Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!

To check that everything works fine launch `helm version` and check that client and tiller versions are the same. In this case you are ready to create your Kubernetes application using Helm.

> if you need to update Tiller launch `helm init upgrade`