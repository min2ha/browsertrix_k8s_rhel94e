# browsertrix_k8s_rhel94e


# RedHat 9.4 Enterprise, Server

```
cat /etc/os-release

NAME="Red Hat Enterprise Linux"
VERSION="9.4 (Plow)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="9.4"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Red Hat Enterprise Linux 9.4 (Plow)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:9::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_BUGZILLA_PRODUCT_VERSION=9.4
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.4"
```


# Register and automatically subscribe in one step

```
subscription-manager register --username <username> --password <password> --auto-attach
```


# Preparation

# (ALL NODES)

## Disable swap space
```
sudo swapoff -a
```

comment in /etc/fstab


## Disable SELinux
Since kubelet has improvable support for SELinux, the upstream documentation recommends to set it to permissive mode (which basically disables SELinux).

```
# Set SELinux to permissive mode
setenforce 0

# Make setting persistent
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

```



## Letting iptables see bridged traffic
### (ALL NODES) Enabling “overlay” and “br_netfilter” kernel modules-

```
cat <<EOF > /etc/modules-load.d/crio-net.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter


cat <<EOF > /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```


## Master node

```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --reload
```

## Worker node

```
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp   
sudo firewall-cmd --reload
```



# Install Kubernetes
https://kubernetes.io/docs/setup/
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


## Steps:


2 Configure yum repo for dvd and permanently mount it in /etc/rc.local
```
  yum install net-tools vim -y
```


## CRI-O Container Engine

The CRI-O container engine provides a stable, more secure, and performant platform for running Open Container Initiative (OCI) compatible runtimes. You can use the CRI-O container engine to launch containers and pods by engaging OCI-compliant runtimes like runc, the default OCI runtime, or Kata Containers. CRI-O’s purpose is to be the container engine that implements the Kubernetes Container Runtime Interface (CRI) for OpenShift Container Platform and Kubernetes, replacing the Docker service.

CRI-O is meant to provide an integration path between OCI conformant runtimes and the Kubelet. Specifically, it implements the Kubelet Container Runtime Interface (CRI) using OCI conformant runtimes. The scope of CRI-O is tied to the scope of the CRI.

At a high level, we expect the scope of CRI-O to be restricted to the following functionalities:

- Support multiple image formats including the existing Docker image format
- Support for multiple means to download images including trust & image verification
- Container image management (managing image layers, overlay filesystems, etc)
- Container process lifecycle management
- Monitoring and logging required to satisfy the CRI
- Resource isolation as required by the CRI


(References: https://github.com/cri-o/cri-o, https://github.com/cri-o/cri-o/blob/main/install.md)

Issues with CRI-O installation on rhel 9: https://github.com/cri-o/cri-o/issues/5905

https://kubernetes.io/blog/2023/10/10/cri-o-community-package-infrastructure/


Add the Kubernetes repo - choose one of two below

### Version 1.28

```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF
```

### Version 1.30

```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```

### Add the CRI-O repo 

Current CRI-O prerelease version was 1,31

```
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/rpm/repodata/repomd.xml.key
EOF
```

Install official package dependencies 
```
dnf install -y \
    conntrack \
    container-selinux \
    ebtables \
    ethtool \
    iptables \
    socat
```

Install the packages from the added repos 
```
dnf install -y --repo cri-o --repo kubernetes \
    cri-o \
    kubeadm \
    kubectl \
    kubelet
```

```
Installed:
  cri-o-1.31.0~dev-150500.73.1.x86_64       cri-tools-1.30.0-150500.1.1.x86_64       kubeadm-1.30.2-150500.1.1.x86_64            
  kubectl-1.30.2-150500.1.1.x86_64          kubelet-1.30.2-150500.1.1.x86_64         kubernetes-cni-1.4.0-150500.1.1.x86_64      

```

# Start CRI-O 
```
systemctl start crio.service
```

### Jump to step 11 

TODO: alternatives CRI-O 
- ContainerD
- Docker

<s>like this
 --
3 configure yum repo for docker
```
  vim /etc/yum.repos.d/docker.repo
  [docker]
  baseurl=https://download.docker.com/linux/cen...
  gpgcheck=0
```
--

4 install docker for rhel8 --nobest and enable docker
```
  yum install docker-ce --nobest -y
```


10 (if docker used only!) 
docker change cgroup driver to systemd, FOR getting logs from containers
   ```
   ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
   ```

   If above command does not work than use this
   refer github doc
   https://github.com/kubernetes/kubeadm...
   
   ```
   #Restarting docker service
   systemctl daemon-reload
   systemctl restart docker
   ```




11 (TODO: check if its not overlapping)
   ```
   yum install -y iproute-tc
   ```
   useful package for vm connectivity

</s>


12 Disabling swap in ram
   ```
   vim /etc/fstab 
   #comment swap line
   ```

13 Extra check: mac addess and product_uuid should be different for every node
   ```
   cat /sys/class/dmi/id/product_uuid
   ```
   the value of product_uuid and mac should be different in all nodes




# Kubernetes cluster setup
```
systemctl enable --now crio && systemctl enable --now kubelet
```

```
# Make sure to reboot to have layered packages available
systemctl reboot
```

## Kubernetes cluster init

Since CRI-O was chosen as the container runtime the correct cgroup driver to be used by kubelet has to be set.
```
echo "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd" | tee /etc/sysconfig/kubelet

```


Put correct IP of your master node, for example 192.168.121.51
```
 sudo kubeadm init --control-plane-endpoint="192.168.121.51:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
# Kubernetes cluster Reset

```
 sudo kubeadm reset
 ```

# Prepare to start using cluster
To start using your cluster, you need to run the following as a regular user:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Log off, log in (in case of terminal)

```
kubectl get pods --all-namespaces
kubectl get all --all-namespaces -o wide
```

## Install CNI
### Calico
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

Install Calico

https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

1.
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

```
2.
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

```

Watch and Wait
```
watch kubectl get pods -n calico-system
```

Check  
```
root@kmasterrhel91 vagrant]# kubectl get nodes -o wide
NAME                        STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
kmasterrhel91.example.com   Ready    control-plane   19m   v1.28.11   192.168.121.51   <none>        Red Hat Enterprise Linux 9.4 (Plow)   5.14.0-427.22.1.el9_4.x86_64   cri-o://1.31.0
```


# Install Snap

Snap will be used for kubectl and HELM installation.

The EPEL repository can be added to RHEL 9 with the following command:
```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf upgrade
```
Adding the optional and extras repositories is also recommended (if errors nevermind):
```
sudo subscription-manager repos --enable "rhel-*-optional-rpms" --enable "rhel-*-extras-rpms"
sudo yum update
```

```
 sudo yum install snapd
 sudo systemctl enable --now snapd.socket
 sudo ln -s /var/lib/snapd/snap /snap

```
Log off, log in (in case of terminal)


# Install kubectl 
## (only if needed!)

Snap could be used to install kubectl too
```
sudo snap install kubectl --classic
```

# Install HELM
Snap will be used to install HELM


```
sudo snap install helm --classic
```

(Reference https://snapcraft.io/install/helm/rhel)


# Deploy Browsertrix using HELM

https://docs.browsertrix.com/deploy/local/
```
helm repo add browsertrix https://docs.browsertrix.com/helm-repo/
helm upgrade --install btrix browsertrix/browsertrix
```

