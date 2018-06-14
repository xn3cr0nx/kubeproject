# Kubernetes The Kubespray Way

Kubespray is a tool that permits to create production ready Kubernetes cluster using under the hood a series of Ansible scripts.

## Set up machines

We need to create a bunch of machines we'll provision via _ansible_

> Instances need at least 1.5gb

* Create instances, I'm going to use do this time to easy ssh into them:

```bash
for i in 0 1 2; do
	doctl compute d create machine-${i} --size 2gb --image ubuntu-16-04-x64 --region lon1 --ssh-keys 4f:f4:41:64:3d:25:90:c6:ab:0e:c7:38:c6:17:ec:1b
done
```

* Every machine need python, minimal is enough:

```bash
for i in 0 1 2; do
  ssh root@$(doctl compute d list --format "Name,PublicIPv4" | grep machine-${i} | awk '{print $2}') 'apt install python-minimal -y'
done
```

## Prerequisites

We need some tools kubespray uses under the hood to provision the cluster

> Python3 seems to not work with these packages, use pip2 to install

* Ansible >= v2.3 `pip2 install ansible` or

```bash
sudo apt-get update
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

* Jinja >= 2.9 `pip2 install jinja2 --upgrade`
* Check IPv4 forwarding is enabled: `sudo sysctl net.ipv4.ip_forward`
* If not enable it: `sudo sysctl -w net.ipv4.ip_forward=1`
* Install kubespray cli: `pip2 install kubespray`
* If during the installation some package fails use `python -m easy_install package_name` to install it

## Setup inventory

Ansible inventory is a file that describes cluster's machines and their roles. Here we have three nodes

* Create new inventory file at _~/.kubespray/inventory/inventory.cfg_:

```ansible
instance-0 ansible_ssh_host=35.189.209.88 ansible_ssh_user=root
instance-1 ansible_ssh_host=35.187.160.64 ansible_ssh_user=root
instance-2 ansible_ssh_host=35.195.51.67 ansible_ssh_user=root

[kube-master]
instance-0
instance-1

[etcd]
instance-0
instance-1
instance-2

[kube-node]
instance-1
instance-2

[k8s-cluster:children]
kube-node
kube-master
```

* Check if it is possible to reach the machines `ansible -i .kubespray/inventory/inventory.cfg -m ping machine-0`
  > You should receive pong as response

## Deployment

Kubespray will install kubernetes-api-server, etcd (key-value store), controller, Scheduler will be installed on master machines and kubelet, kube-proxy and Docker (or rkt) will be installed on node machines (minions). These all components will be installed and configured by ansible roles in kubespray. All, We need to do is to execute one command.

* Start deploying: `kubespray deploy`
  > This process takes really long, up to 20 minutes

## Connect kubectl to the cluster

You need to set the context and have the certificates on your client to communicate with the cluster

* Get the cert from master:

```bash
mkdir kubectl
ssh root@master1_ip sudo cat /etc/kubernetes/ssl/admin-machine-0.pem > kubectl/admin-machine-0.pem
ssh root@master1_ip sudo cat /etc/kubernetes/ssl/admin-machine-0-key.pem > kubectl/admin-machine-0-key.pem
ssh root@master1_ip sudo cat /etc/kubernetes/ssl/ca.pem > kubectl/ca.pem
```

> get ip of the instance with this command and substitute to master1_ip `$(doctl compute d ls | grep master_1 | awk '{print $3}')` to  `master1_ip`

* Configure kubectl:

```bash
kubectl config set-cluster kubespray --server=https://master1_ip:6443 --certificate-authority=kubectl/ca.pem

kubectl config set-credentials kadmin \
    --certificate-authority=kubectl/ca.pem \
    --client-key=kubectl/admin-machine-0-key.pem \
    --client-certificate=kubectl/admin-machine-0.pem

kubectl config set-context kubespray --cluster=kubespray --user=kadmin
kubectl config use-context kubespray

kubectl get node
kubectl get all --all-namespaces
```

> Check from the master that the secure port for the api is _6443_ with `kubectl describe po/kube-apiserver-machine-0`

test sample application `kubectl create namespace sock-shop`
`kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"`
