# Kubernetes The Kubeadm Way

The entire kubernetes the hard way setup can be automated trough scripts. Kubeadm aimed this result setting great part of the provisioning from the hard way with just some commands.
kubeadm is a toolkit that helps you bootstrap a best-practice Kubernetes cluster in an easy, reasonably secure and extensible way.
It by design does not install a networking solution for you, which means you have to install a third-party CNI-compliant networking solution yourself using kubectl apply.
kubeadm aims to set up a minimum viable cluster that pass the Kubernetes Conformance tests, but installing other addons than really necessary for a functional cluster is out of scope.

## Setting the environment
The kubeadm tool requires the following machines running one of Ubuntu 16.04+, HypriotOS v1.0.1+, or CentOS 7 running on them: 
-	One machine for the master node
-	One or more machines for the worker nodes

In this case I'm going to set a machine for the master and two nodes for workers.
Each instances should be created and setted up as for the hard way doc, this means the Provisioning section.
- Create VPC, subnet and firewall-rules as before
- Create master: 
```
gcloud compute instances create master \
  --async \
  --boot-disk-size 200GB \
  --can-ip-forward \
  --image-family ubuntu-1604-lts \
  --image-project ubuntu-os-cloud \
  --machine-type n1-standard-1 \
  --private-network-ip 10.240.0.10 \
  --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
  --subnet kubernetes \
  --tags kubernetes-the-hard-way,controller
```
- Create workers: 
```
for i in 1 2; do
  gcloud compute instances create worker-${i} \
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
    --tags kubernetes-the-hard-way,worker
done
```

- Install Docker, kubelet, kubectl, and kubeadm on each of the three machines as root `sudo su`.
	- Docker: `curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh`
	- Add user to docker sudo group: `usermod -aG docker <user>`
	- kubectl kubelet kubeadm:
		- `apt-get update && apt-get install -y apt-transport-https curl`
		- `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`
		- ```
			cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
			deb http://apt.kubernetes.io/ kubernetes-xenial main
			EOF
			```
		- `apt-get update && apt-get install -y kubelet kubeadm kubectl`
	- On Master Node edit /etc/systemd/system/kubelet.service.d/10-kubeadm.conf to match docker cgroup: `nano /etc/systemd/system/kubelet.service.d/10-kubeadm.conf`
	- Add to a config `--cgroup-driver=cgroupfs`
	- Restart kubelet: `systemctl daemon-reload` `systemctl restart kubelet` 


## Provisioning
Keep the root previlegies and start the provisioning through kubeadm

- Initializing the master: `kubeadm init --pod-network-cidr=192.168.0.0/16`
> You have to add the flag to use calico CNI

The master will return the command needed to join the cluster by the worker, containing the token required.
This is what I got back
- Export kubeconfig as normal user:
	- `sudo cp /etc/kubernetes/admin.conf $HOME/`
	- `sudo chown $(id -u):$(id -g) $HOME/admin.conf`
	- `export KUBECONFIG=$HOME/admin.conf`
	> I had to add the KUBECONFIG variable to the /etc/environment file because every time the export was deleted
- Export kubeconfig as root: `export KUBECONFIG=/etc/kubernetes/admin.conf`
- Install pod network (Calico): `kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml`
- Check Calico pods creation is on Running: `kubectl get pods --all-namespaces`
- Check nodes status (now they should be Ready): `kubectl get nodes`
- Remove taint from master to able it to schedule pod: `kubectl taint nodes --all master-`
- Use kube join as above specified to join workers to the cluster: `kubeadm join 10.240.0.10:6443 --token dvlqtf.kqnmzsg8buofeodq --discovery-token-ca-cert-hash sha256:4e68bf2cdd0c7826673d6c2f6b545e8e71d6db5a86a5dcaf68c12ede3f8c7020`

So that's it! Everything here is a summary of what we did in the hard way path. So easier

- To use kubectl from outside the master: `scp root@<master ip>:/etc/kubernetes/admin.conf .` 
`kubectl --kubeconfig ./admin.conf get nodes`

Notice that with kubeadm you create a single master, that became the single point of failure of the cluster


