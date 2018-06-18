# Deploying WordPress with Helm

![Kubernetes WordPress](https://cdn.deliciousbrains.com/content/uploads/2017/08/10131149/db-WPKubernetesCluster-1540x748.jpg)

## RBAC fix

As I introduced in the Helm chapter, you installed Helm client from script and then Tiller server launching `helm init`.

Now you need to implement a RBAC fix to be able to deploy your application. RBAC is for Role-based access control, so we are talking about permissions.

To give Tiller permission to install and configure resources on our cluster, first we create a new ServiceAccount named tiller in the system-wide kube-system Namespace.  
`kubectl create serviceaccount --namespace kube-system tiller`

Create a ClusterRoleBinding with cluster-admin access levels for this new service account named tiller. Note that these are very broad permissions, because we want Tiller to be able to install and configure a wide variety of cluster resources.
`kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`

Edit the existing Tiller Deployment named tiller-deploy.  
`KUBE_EDITOR="nano" kubectl edit deploy --namespace kube-system tiller-deploy`  
This will open the YAML configuration for this deployment in nano. Insert the line `serviceAccount: tiller`  
in the spec: template: spec section of the file

## Installing WordPress

Finally we can deploy our application using Helm. We are going to create all pieces of our application in a specific namespace:  
`kubectl create namespace wordpress`

Here comes the magic:  
`helm install -n wordpress --name wordpress --set serviceType=NodePort stable/wordpress`

That's it, the applications need a couple of minutes to bootstrap, but in this way Helm will create wordpress pod, deploy and service based on MariaDB for backend. Amazing!

To visit the website you need to catch the ip:  
`export NODE_PORT=$(kubectl get --namespace wordpress -o jsonpath="{.spec.ports[0].nodePort}" services wordpress-wordpress)`  
`export NODE_IP=$(kubectl get nodes --namespace wordpress -o jsonpath="{.items[0].status.addresses[0].address}")`  
`echo http://$NODE_IP:$NODE_PORT/admin`

In the end you should have a result like this (obtained with `kubectl get all --all-namespaces`:
![Final Result](/assets/final_result.png)

## Scaling

Scaling your microservices is really easy. For example:  
`kubectl scale --replicas 2 deployments wordpress-wordpress -n wordpress`  
With this command you can scale up to 2 replicas for the number of deployments of wordpress-wordpress deployment with just a command