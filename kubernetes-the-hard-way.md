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

- Create three compute instances which will host the Kubernetes control plane Controllers
```
for i in 0 1 2; do
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
  done
```

- Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range
```
for i in 0 1 2; do
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

## Components
In the next chapter I'm going to creates certificates for etcd, kube-apiserver, kubelet, and kube-proxy. But it would be better to first understand what these components are.

#### Master Components
They provide the cluster's control plane, they make global decision about the cluster and detecting and responding to cluster events
- kube-apiserver: component on the master that expos the k8s API ("frontend" for k8s control plane)
- etcd: consistent and high-available key value store used as k8s' backing store for all cluster data
- kube-scheduler: component on the master that watches newly created pods that have no node assigned and select a node for them to run on
- kube-controller-manager: component on the master that runs controllers
(*controllers*: a control loop that watched the shared state of the cluster though the apiserver and makes changes attempting to move the current state toward teh desired state)
  - Node Controller: responsible for noticing and responding when nodes go down
  - Replication Controller: responsible for maintaining the correct number of pods for every replication controller object in the system
  - Endpoints Controller: populates the Endpoints object (joins Services & Pods)
  - Service Account & Token Controller: creates default accounts and API access token for new namespaces
- cloud-controller-manager: runs controller that interacts with the underlying cloud providers
  - Node Controller: for checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding
  - Router Controller: for setting up routes in the underlying cloud infrastructure
  - Service Controller: for creating, updating and deleting cloud provider load balancers
  - Volume Controller: for creating, attaching, and mounting volumes, and interacting with the cloud provider to orchestrate volumes

#### Node Components
They run on every node, maintaining running pods and providing the k8s runtime environment
- kubelet: it makes sure that containers are running in a pod
- kube-proxy: enables the k8s service abstraction by maintaining network rules on the host and performing connection forwarding
- Container Runtime: software responsible for running containers (eg. Docker, rkt..)

#### Addons
Pods and services that implement cluster features. Namespaced addon objects are created in the kube-system namespace
- DNS: all k8s clusters should have cluster DNS. Cluster DNS is a DNS server, in addition to the other DNS server(s) in your environment, which serves DNS records for Kubernetes services
- Web UI (Dashboard):  general purpose, web-based UI for k8s clusters
- Container Resource Monitoring: records generic time-series metrics about containers in a central database, and provides a UI for browsing that data
Cluster-level Logging: is responsible for saving container logs to a central log store with search/browsing interface. 


## Provisioning a CA and Generating TLS Certificates
A public key infrastructure (PKI) is a set of roles, policies, and procedures needed to create, manage, distribute, use, store, and revoke digital certificates and manage public-key encryption.

Provision a CA that can be used to generate additional TLS certificates
- Create CA configuration file: 
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```
- Create the CA certificate signing request:
```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```
- Generate the CA certificate and private key `cfssl gencert -initca ca-csr.json | cfssljson -bare ca`
- Create the admin client certificate signing request:
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```
- Generate the admin client certificate and private key:
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```
- Generate a certificate and private key for each Kubernetes worker node:
```
for instance in worker-0 worker-1 worker-2; do
  cat > ${instance}-csr.json <<EOF
  {
    "CN": "system:node:${instance}",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "US",
        "L": "Portland",
        "O": "system:nodes",
        "OU": "Kubernetes The Hard Way",
        "ST": "Oregon"
      }
    ]
  }
  EOF
  EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
    --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
  INTERNAL_IP=$(gcloud compute instances describe ${instance} \
    --format 'value(networkInterfaces[0].networkIP)')
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
    -profile=kubernetes \
    ${instance}-csr.json | cfssljson -bare ${instance}
  done
```
API requests made by kubelet are authorized by a special-purpose authorization mode called *Node Authorizer*. In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the system:nodes group, with a username of system:node:`<nodeName>`

- Create the kube-proxy client certificate signing request:
```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```
- Generate the kube-proxy client certificate and private key: `cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy`
- Retrieve the kubernetes-the-hard-way static IP address: `KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way --region $(gcloud config get-value compute/region) --format 'value(address)')`
- Create the Kubernetes API Server certificate signing request:
```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```
- Generate the Kubernetes API Server certificate and private key:
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
- Copy the appropriate certificates and private keys to each worker instance:
```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```
- Copy the appropriate certificates and private keys to each controller instance:
```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem ${instance}:~/
done
```

## Generating Kubernetes Configuration Files for Authentication
- Retrieve public static IP address: `KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses list | grep kubernetes-the-hard-way | awk '{print $3}' | sed -n 2p)`
- Generate a kubeconfig file for each worker node (kubelet config file):
```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
- Generate a kubeconfig file for the kube-proxy service:
`kubectl config set-cluster kubernetes-the-hard-way --certificate-authority=ca.pem --embed-certs=true --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 --kubeconfig=kube-proxy.kubeconfig`

`kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem  --embed-certs=true --kubeconfig=kube-proxy.kubeconfig`

`kubectl config set-context default --cluster=kubernetes-the-hard-way --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig`

`kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig` 

- Copy the appropriate kubelet and kube-proxy kubeconfig files to each worker instance:
```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

## Generating the Data Encryption Config and Key
- Generate an encryption key: `ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)`
- Create the encryption-config.yaml encryption config file:
```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
- Copy the encryption-config.yaml encryption config file to each controller instance:
```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

## Bootstrapping the etcd Cluster
To Login in each instance use `gcloud compute ssh ${instance}`
For each controller:
- Download the official etcd release binaries from the coreos/etcd GitHub project: `wget -q --show-progress --https-only --timestamping "https://github.com/coreos/etcd/releases/download/v3.2.11/etcd-v3.2.11-linux-amd64.tar.gz"`
- Extract etcd: `tar -xvf etcd-v3.2.11-linux-amd64.tar.gz`
- Install the etcd server and the etcdctl cli: `sudo mv etcd-v3.2.11-linux-amd64/etcd* /usr/local/bin/`
- Creates system folder for the server: `sudo mkdir -p /etc/etcd /var/lib/etcd`
- Configure the etcd server: `sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/`
- Retrive the internal IP address this way: `INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)`
- Set the etcd name to match the hostname of the current compute instance (each etcd member must have a unique name within the etcd cluster): `ETCD_NAME=$(hostname -s)`
- Create the etcd.service systemd unit file: 
```
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Start the etcd server: 
  - `sudo mv etcd.service /etc/systemd/system/`
  - `sudo systemctl daemon-reload`
  - `sudo systemctl enable etcd`
  - `sudo systemctl start etcd`

- From a controller check the cluster is built: `ETCDCTL_API=3 etcdctl member list`


## Bootstrapping the Kubernetes Control Plane
Installing Kubernetes API Server, Scheduler, and Controller Manager inside each node. I'm going to configure an external LB too that will expose the k8s API server to remote clients
For each controller:
- Download the official Kubernetes release binaries:
```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl"
```
- Install the Kubernetes binaries: `chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl &&
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/`
- Configure the Kubernetes API Server: `sudo mkdir -p /var/lib/kubernetes/ && sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem encryption-config.yaml /var/lib/kubernetes/`
- Retrieve the internal IP address: `INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)`
- Create the kube-apiserver.service systemd unit file:
```
cat > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --insecure-bind-address=127.0.0.1 \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Configure the Kubernetes Controller Manager creating the kube-controller-manager.service systemd unit file:
```
cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Configure the Kubernetes Scheduler creating the kube-scheduler.service systemd unit file:
```
cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Start the Controller Services: `sudo mv kube-apiserver.service kube-scheduler.service kube-controller-manager.service /etc/systemd/system/`
  - `sudo systemctl daemon-reload`
  - `sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler`
  - `sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler`

- Verify the status of components: `kubectl get componentstatuses`

### RBAC for Kubelet Authorization
Configuring RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. This is required for retrieving metrics, logs, and executing commands in pods.
For each controller:
- Create the *system:kube-apiserver-to-kubelet* **ClusterRole**:
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```
- Bind the system:kube-apiserver-to-kubelet ClusterRole to the kubernetes user:
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### The Kubernetes Frontend Load Balancer
Provisioning an external load balancer to front the Kubernetes API Servers
- Create the external load balancer network resources: 
  - `gcloud compute target-pools create kubernetes-target-pool`
  - `gcloud compute target-pools add-instances kubernetes-target-pool --instances controller-0,controller-1,controller-2`
  - `KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way
 --region $(gcloud config get-value compute/region) --format 'value(name)')`
  - `gcloud compute forwarding-rules create kubernetes-forwarding-rule \
  --address ${KUBERNETES_PUBLIC_ADDRESS} \
  --ports 6443 \
  --region $(gcloud config get-value compute/region) \
  --target-pool kubernetes-target-pool`
- Verify the successful of operation:
  - Retriev the public IP: `KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')`
  - Make a HTTP request for the Kubernetes version info: `curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version`


## Bootstrapping the Kubernetes Worker Nodes
The following components will be installed on each node: runc, container networking plugins, cri-containerd, kubelet, and kube-proxy.
For each worker:
- Install the OS dependencies: `sudo apt-get -y install socat`
Socat  is a command line based utility that establishes two bidirectional byte streams and transfers data between them. The socat binary enables support for the kubectl port-forward command.
- Download worker binaries: 
```
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/containerd/cri-containerd/releases/download/v1.0.0-beta.1/cri-containerd-1.0.0-beta.1.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubelet
```
- Create the installation directories: 
```
sudo mkdir -p \
/etc/cni/net.d \
/opt/cni/bin \
/var/lib/kubelet \
/var/lib/kube-proxy \
/var/lib/kubernetes \
/var/run/kubernetes
```
- Install the worker binaries: 
  - `sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/`
  - `sudo tar -xvf cri-containerd-1.0.0-beta.1.linux-amd64.tar.gz -C /`
  - `chmod +x kubectl kube-proxy kubelet`
  - `sudo mv kubectl kube-proxy kubelet /usr/local/bin/`

- Configure CNI Networking retrieving the Pod CIDR range for the current compute instance: `POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)`
- Create the bridge network configuration file:
```
cat > 10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```
- Create the loopback network configuration file:
```
cat > 99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```
- Move the network configuration files to the CNI configuration directory: `sudo mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/`
- Configure the Kubelet:
  - `sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/`
  - `sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig`
  - `sudo mv ca.pem /var/lib/kubernetes/`
- Create the kubelet.service systemd unit file:
```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cloud-provider= \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --pod-cidr=${POD_CIDR} \\
  --register-node=true \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.pem \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Configure the Kubernetes Proxy: `sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig`
- Create the kube-proxy.service systemd unit file:
```
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.200.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- Start the Worker Services:
  - `sudo mv kubelet.service kube-proxy.service /etc/systemd/system/`
  - `sudo systemctl daemon-reload`
  - `sudo systemctl enable containerd cri-containerd kubelet kube-proxy` 
  - `sudo systemctl start containerd cri-containerd kubelet kube-proxy`
- Verify procedure, inside a controller: `kubectl get nodes`

## Configuring kubectl for Remote Access
Generating a kubeconfig file for the kubectl command line utility based on the admin user credentials.
- Retrieve the kubernetes-the-hard-way static IP address: `KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')`
- Generate a kubeconfig file suitable for authenticating as the admin user:
  - ```
      kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.pem \
      --embed-certs=true \
      --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
    ```
  - ```
    kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem
    ```
  - ```
    kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin
    ```
  - `kubectl config use-context kubernetes-the-hard-way`

- Verify is all set:
  - `kubectl get componentstatuses`
  - `kubectl get nodes`

## Provisioning Pod Network Routes
CIDR (Classless Inter-Domain Routing): is a method for allocating IP addresses and IP routing
Pods scheduled to a node receive an IP from node's Pod CIDR range, but they cannot communicate with other pods running on other node due to missing network routes.
A route is a mapping of an IP range to a destination. Routes tell the VPC network where to send packets destined for a particular IP address.
The third chapter Implementations explains all the ways to implement a network route.
- To gather the information required to create routes in the kubernetes-the-hard-way VPC network print the internal IP address and Pod CIDR range for each worker instance:
```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done
```
The Routing Table
- Create network routes for each worker instance:
```
for i in 0 1 2; do
  gcloud compute routes create kubernetes-route-10-200-${i}-0-24 \
    --network kubernetes-the-hard-way \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done
```
- List the routes in the kubernetes-the-hard-way VPC network: `gcloud compute routes list --filter "network: kubernetes-the-hard-way"`

## Deploying the DNS Cluster Add-on
Deploying the DNS add-on which provides DNS based service discovery to applications running inside the Kubernetes cluster.

- Deploy the kube-dns cluster add-on: `kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml`
The output will be 
serviceaccount "kube-dns" created
configmap "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created
This pods will be created in the kube-system namespace, so you have to specify the namespace to show them
- List them: `kubectl get pods -l k8s-app=kube-dns -n kube-system`
Verify DNS is set up:
- Create a busybox deployment: `kubectl run busybox --image=busybox --command -- sleep 3600`
- List the pod created by the busybox deployment: `kubectl get pods -l run=busybox`
- Retrieve the full name of the busybox pod: `POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")`
- Execute a DNS lookup for the kubernetes service inside the busybox pod: `kubectl exec -ti $POD_NAME -- nslookup kubernetes`

## Smoke Test
Verifying to encrypt secret data at rest, and other main k8s funcionalities

#### Data Encryption
- Create generic secret: `kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"`
A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key.
- Print a hexdump of the kubernetes-the-hard-way secret stored in etcd: `gcloud compute ssh controller-0 --command "ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"`

#### Deployments
- Create a deployment for the nginx web server: `kubectl run nginx --image=nginx`

#### Port Forwarding
- Retrieve the full name of the nginx pod: `POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")`
- Forward port 8080 on your local machine to port 80 of the nginx pod: `kubectl port-forward $POD_NAME 8080:80`
- Make an HTTP request using the forwarding address: `curl --head http://127.0.0.1:8080`

#### Logs
- Print the nginx pod logs: `kubectl logs $POD_NAME`


#### Exec
- Print the nginx version by executing the nginx -v command in the nginx container: `kubectl exec -ti $POD_NAME -- nginx -v`

#### Services
- Expose the nginx deployment using a NodePort service: `kubectl expose deployment nginx --port 80 --type NodePort`
- Retrieve the node port assigned to the nginx service: `NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')`
- Create a firewall rule that allows remote access to the nginx node port: `gcloud compute firewall-rules create kubernetes-the-hard-way-allow-nginx-service --allow=tcp:${NODE_PORT} --network kubernetes-the-hard-way`
- Retrieve the external IP address of a worker instance: `EXTERNAL_IP=$(gcloud compute instances describe worker-0 
--format 'value(networkInterfaces[0].accessConfigs[0].natIP)')` this will return the external IP of the worker where the pod is running
- Make an HTTP request using the external IP address and the nginx node port: `curl -I http://${EXTERNAL_IP}:${NODE_PORT}` In the browser you will find the welcome to nginx! page