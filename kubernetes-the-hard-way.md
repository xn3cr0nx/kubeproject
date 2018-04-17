# Networking
k8s assumes pods can communicate with other pods, regardless of which host they land on

### Docker model
"normal" -> host-private networking: virtual bridge called docker0 and allocates a subnet from one of the private address blocks defined in RFC1918 (Address Allocation for Private Internets).
Virtual Ethernet device (veth) for each container which is attached to the bridge. Veth is mapped to appears as eth0 in the container using Linux namespaces and is given an IP address from the bridge's address range ==> containers can talk to other containers only if they are on the same machine(same virtual bridge).
To comunicate across nodes, there must be allocated ports on the machine's own IP address, which are the forwarded or proxied to the containers.

### Kubernetes model
Imposes fundamental requirements on any networking implementation:
  - all containers(Pods) can communicate with all other containers(Pods) without NAT
  - all nodes can communicate with all containers(Pods) (and vice-versa) without NAT
  - the IP that a container(Pod) sees itself as is the same IP that others see it as
Containers within a Pod share their network namespaces including IP address => "IP-per-pod" model
Docker is a "pod container" which holds the network namespace open while "app containers" join that namespace with Docker's --net=container:<id> function

### Implementations (A lot of solutions)
SDN = software-defined networking, novel approach to cloud computing that facilitates network management
- ACI: (Cisco Apoplication Centric Infrastructure) intergrated overlay and undelay SDN solution that supports containers, VMs and bare metal servers
- Big Cloud Fabric from Big Switch Networks: cloud native networking architecture, designed to run Kubernetes in private cloud/on-premise environments. With this is possible to connect container orchestration system (k8s, openshift, swarm..) along side with VM orchestration system (VMware, openstack..)
- Cilium: open source software for providing and transparently securing network connectivity between application containers.
- Contiv: provides configurable networking for variuous use cases and open source
- Contrail: truly open multi-cloud network virtualization and policy management platform
- Flannel: simple overlay network that satisfies the K8S requirements
- GCE: (Google Compute Engine) The result of all this (a lot of shit wrote above) is that all Pods can reach each other and can egress traffic to the internet
- Kube-Router: purpose-built networking solution for K8S that aims to provide high performance and operational simplicity
- L2 networks and linux bridging: you can do something like GCE (geek geek geek)
- Multus: multi CNI plugin to support multi networking feature in k8s using CRD based network objects in k8s (multi supports eg Flannel, DHCP, Macvlan) and 3rd party plugins (Calico, Weave Cilium, Contiv)
- VMware NSX-T: network virtualization and security platform can provide network virtualization for multi-cloud and multi-hypervisor environment, focus on emerging application frameworks and architectures with heterogeneous endpoints and technology stacks
- Nuage Networks VCS: (Virtualized Cloud Services) highly scalable policy-based software-defined networking (SDN) platform. Uses overlays to provide seamless policy-based networking between K8S Pods and non-Kubernetes environements (VMs and bare metal servers)
- OperVSwitch: more mature but also complicated way to build an overlay network
- OVN: (Open Virtual Networkin) open source network virtualization solution that lets one create logical switches, logical routes, stateful ACLs, LBs etc to build different virtual networking topologies
- Calico: open source container networking provider and network policy engine. Provies higly scalable networking and network policy solution for connecting K8S pods based on the same IP networking principles as the internet.
- Romana: open source network and security automation solution that lets you deploy K8S without an overlay network
- Weave Net from Weaveworks: resilient and simple to use network for K8S and its hosted applications
- CNI-Genie from Huawei: plugin that enables K8S to simultaneously have access to different implementations of the K8S network model in runtime (CNI-plugin, Flannel, Calico, Romana, Weave-net) and supports assigning multiple IP addresses to a pod, each from a different CNI plugin

### Network Policies
Specification of how groups of pods are allowd to communicate with each other and other network endpoints
NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to the selected pods.
Are implemented by the network plugin
When a NetworkPolicy select a particular pod, that pod will reject any connections that are not allowed by any NetworkPolicy, the pod becames isolated. By default they're non-isolated, accepting traffic from any source.


### Virtual Private Cloud Network
You should create VPC Newtork, in DO this doesn't seam to be possible
Start using Google Cloud Engine

# Kubernetes The Hard Way
## Provisioning
- confgure your project on GCE (good luck)
- create a VPC Newtork `gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom`
- create kubernetes subnet in the VPC `gcloud compute networks subnets create kubernetes --network kubernetes-the-hard-way --range 10.240.0.0/24`
- create a firewall rule that allows internal communication across all protocols `gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal --allow tcp,udp,icmp --network  kubernetes-the-hard-way --source-ranges 10.240.0.0/24,10.200.0.0/16`
- Create a firewall rule that allows external SSH, ICMP, and HTTPS `gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external --allow tcp:22,tcp:6443,icmp --network kubernetes-the-hard-way --source-ranges 0.0.0.0/0`
- Allocate static IP that will be attached to the external LB fronting the k8s API servers: `gcloud compute addresses create kubernetes-the-hard-way --region $(gcloud config get-value compute/region)`
 - Remember you should have a compute/region set, if not use `gcloud config set compute/region europe-west1` && `gcloud config set compute/zone us-west1-a` for the zone

- Create three compute instances which will host the Kubernetes control plane `Controllers`
 `for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
  done`
  `gcloud compute instances create controller-0 --async --boot-disk-size 200GB --can-ip-forward --image-family ubuntu-1604-lts --image-project ubuntu-os-cloud --machine-type n1-standard-1 --private-network-ip 10.240.0.10 --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring --subnet kubernetes --tags kubernetes-the-hard-way,controller`

- Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range
  `for i in 0 1 2; do
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
done`

## Provisioning a CA and Generating TLS Certificates
A public key infrastructure (PKI) is a set of roles, policies, and procedures needed to create, manage, distribute, use, store, and revoke digital certificates and manage public-key encryption.

Provision a CA that can be used to generate additional TLS certificates




















