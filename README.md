# kubernetes
setup kubernetes instructions

Steps to install K8s on VMs:
Each VM for kubernetes cluster requires following configuration:
1. OS: CentOS 7+
2. All VMs must be reachable with an assigned static IP address.
3. User account with sudo access priviledges
4. yum package manager installed

# Installation
## On All Nodes
Kubernetes requires Docker as pre-requisite to function correct.

Steps to install Docker:
If any existing docker installation is present, remove and do fresh install with the latest using same configuration across cluster:
```
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux  docker-engine-selinux docker-engine docker-ce-cli
```

Update yum packages list
```
sudo yum check-update
```

Install dependencies
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```
yum-utils -> installs yum-config-manager
device-mapper-persistent-data & lvm2 -> docker uses a device mapperstorage driver, these dependencies are needed to run the driver

Add Docker repo to yum
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker
```
sudo yum install docker
```

Configure docker to run as service on startup and run, verify status of docker service:
```
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

Now that docker is installed, verify cgroup driver for docker is systemd (this is recommended driver for k8s)
```
docker info
```
If it is not systemd, it can be changed as follows:
```
vi /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
Restart docker service to effect change
```
sudo systemctl restart docker
```

Add k8s repo to yum
```
vi /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

Kubernetes needs following packages installed across all nodes
```
sudo yum install -y kubelet kubeadm kubectl
```
kubelet -> node agent that facilitates functions with cluster (api server) and monitor resources on the node
kubeadm -> admnistration of cluster, node upgrades, join/remove from cluster, certificate rotations, etc
kubectl -> optional, client to communicate with api-server

Enable kubelet as service
```
systemctl enable kubelet
systemctl start kubelet
```

set hostnames on each node, for routing local DNS
```
sudo vi /etc/hosts
10.104.129.187 master-node
10.104.129.83 worker-node1
10.104.129.84 worker-node2
```

Open firewall ports for allowing nodes to communicate on cluster
```
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --reload
```
For master node, additionally, allow the following ports:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --reload
```

Update IPtables in sysctl conf to allow packets to process during filtering and port forwarding:
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

Disable security functions in linux to allow containers to access host filesystem
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Disable swap on OS to allow k8s cluster to manage the resource allocation more effectively
```
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

All the above steps are needed to be performed on all nodes.

## Create kubernetes cluster 

### On master node
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
This uses flannel pod networking solution with CIDR range as mentioned above.
Allow port **8285** through firewall for flannel.

To manage cluster as non sudo user, you need to copy kube config file from admin to user home:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check node health on master as follows:
```
sudo kubectl get nodes
```
see the status as READY

Check flannel pod network is functioning by checking coreDNS is running
```
sudo kubectl get pods -A
```

Type the following command to get service discovery token to join worker node:
```
kubeadm token create --print-join-command
```

### On worker node
copy the join command output from master node and run on worker node VM

# Upgrading nodes
Ref official documentation: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

# External references:
https://phoenixnap.com/kb/how-to-install-kubernetes-on-centos
https://phoenixnap.com/kb/how-to-install-docker-centos-7
