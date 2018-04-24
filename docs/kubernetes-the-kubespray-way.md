# Kubernetes The Kubespray Way

## Set up machines
We need to create a bunch of machine we'll provision via ansible
<!-- ```
for i in 0 1 2; do
  gcloud compute instances create instance-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-kubespray-way,worker
done -->
```
```
for i in 
doctl compute d create machine-0 --size 1gb --image ubuntu-16-04-x64 --region lon1 --ssh-keys 4f:f4:41:64:3d:25:90:c6:ab:0e:c7:38:c6:17:ec:1b
```

## Prerequisites
We need some tools kubespray uses under the hood to provision the cluster
- Ansible >= v2.3 `pip install ansible` or
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```
- Jinja >= 2.9 `pip install jinja2 --upgrade`
- Check IPv4 forwarding is enabled: `sudo sysctl net.ipv4.ip_forward`
- If not enable it: `sudo sysctl -w net.ipv4.ip_forward=1`
- Install kubespray cli: `pip install kubespray`

## Setup inventory
Ansible inventory is a file that describes cluster's machines and their roles. Here we have three nodes
- Create new inventory file at *~/.kubespray/inventory/inventory.cfg*:
```
instance-0 ansible_ssh_host=35.189.209.88
instance-1 ansible_ssh_host=35.187.160.64
instance-2 ansible_ssh_host=35.195.51.67

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

## Deployment
Kubespray will install kubernetes-api-server, etcd (key-value store), controller, Scheduler will be installed on master machines and kubelet, kube-proxy and Docker (or rkt) will be installed on node machines (minions). These all components will be installed and configured by ansible roles in kubespray. All, We need to do is to execute one command.
- Start deploying: `kubespray deploy`