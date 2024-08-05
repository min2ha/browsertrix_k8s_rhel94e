# browsertrix_k8s_rhel94e


# RedHat 9.4 Enterprise, Server

In the beginning of installation users can select to install presets automatically. In this case default very basic presets were selected: development, networking, virtualization. 

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
https://github.com/cri-o/packaging

## Steps:

Configure yum repo for dvd and permanently mount it in /etc/rc.local

(depends on installed preset of packages in the beginning, next command may be redundant, but it won't break anything anyway)
```
  yum install net-tools vim -y
```

## Kubernetes and CRI-O Container Engine

Define versions:

```
KUBERNETES_VERSION=v1.30
CRIO_VERSION=v1.30
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


(References: https://github.com/cri-o/cri-o, https://github.com/cri-o/cri-o, https://github.com/cri-o/cri-o/blob/main/install.md)

Issues with CRI-O installation on rhel 9: https://github.com/cri-o/cri-o/issues/5905

https://kubernetes.io/blog/2023/10/10/cri-o-community-package-infrastructure/

### Add the Kubernetes repository

```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF
```


### Add the CRI-O repository

```
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF
```


Install package dependencies from the official repositories

```
dnf install -y container-selinux
```

Install the packages

```
dnf install -y cri-o kubelet kubeadm kubectl
```

Installation results:

```
Installed:
  conntrack-tools-1.4.7-2.el9.x86_64           cri-o-1.30.3-150500.1.1.x86_64                  cri-tools-1.30.0-150500.1.1.x86_64
  kubeadm-1.30.3-150500.1.1.x86_64             kubectl-1.30.3-150500.1.1.x86_64                kubelet-1.30.3-150500.1.1.x86_64
  kubernetes-cni-1.4.0-150500.1.1.x86_64       libnetfilter_cthelper-1.0.0-22.el9.x86_64       libnetfilter_cttimeout-1.0.0-19.el9.x86_64
  libnetfilter_queue-1.0.5-1.el9.x86_64        socat-1.7.4.1-5.el9_4.2.x86_64
```

## Optionally the Kubernetes repo version could be defined manually too - for example choose one of two below

If not interrested skip and jump to Start CRI-O section Otherwise.


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

### Add the CRI-O repo manually (prerelease for example)

Current CRI-O prerelease version was 1.31

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

Enable crio and kubelet services:

```
systemctl enable --now crio && systemctl enable --now kubelet
```

'Enable' will hook the specified unit into relevant places, so that it will automatically start on boot, or when relevant hardware is plugged in, or other situations depending on what's specified in the unit file.


```
# Make sure to reboot to have layered packages available
systemctl reboot
```

## Kubernetes cluster init

Since CRI-O was chosen as the container runtime the correct cgroup driver to be used by kubelet has to be set.
```
echo "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd" | tee /etc/sysconfig/kubelet

```

```
swapoff -a
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
```

Quickstart for Calico on Kubernetes (CNI part)
https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart


By default, in order to init cluster with defalt params, it's suficient to run command:
```
kubeadm init
```

In order to use Calico CNI, init cluster command needs extra pod-network-cidr param:
```
kubeadm init --pod-network-cidr=192.168.0.0/16
```

(Classless Inter-Domain Routing (CIDR) is an IP address allocation method that improves data routing efficiency on the internet.)


It's possible to specify IP address of master node, for example 192.168.121.51
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

Container Network Interface (CNI) is a framework for dynamically configuring networking resources.


#### Initial state before CNI

(BEWARE! A node without a CNI becomes ready when the CRI on that host picks up the CNI, overlay is not mandatory in Kubernetes.)
If you discovered, that nodes in cluster are in `Ready` state already, you may skip Calico deployment. 

Here is a tutorial that talks about this subject:
https://www.tigera.io/tutorials/?_sf_s=Calico%20Basics

`NotReady` status: 
```
[root@rhel94 root]# kubectl get nodes
NAME       STATUS     ROLES           AGE     VERSION
rhel94     NotReady   control-plane   9m27s   v1.30.3
rhel94-2   NotReady   <none>          7m4s    v1.29.7
rhel94-3   NotReady   <none>          16s     v1.30.3

[root@rhel94 root]# kubectl get nodes -o wide
NAME       STATUS     ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel94     NotReady   control-plane   11m     v1.30.3   192.168.1.109   <none>        Red Hat Enterprise Linux 9.4 (Plow)   5.14.0-427.28.1.el9_4.x86_64   cri-o://1.31.0
rhel94-2   NotReady   <none>          9m36s   v1.29.7   192.168.1.127   <none>        Red Hat Enterprise Linux 9.4 (Plow)   5.14.0-427.28.1.el9_4.x86_64   cri-o://1.31.0
rhel94-3   NotReady   <none>          2m48s   v1.30.3   192.168.1.111   <none>        Red Hat Enterprise Linux 9.4 (Plow)   5.14.0-427.28.1.el9_4.x86_64   cri-o://1.30.4
```

https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

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
root@kmasterrhel91]# kubectl get nodes -o wide
NAME                        STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
kmasterrhel91.example.com   Ready    control-plane   19m   v1.28.11   192.168.121.51   <none>        Red Hat Enterprise Linux 9.4 (Plow)   5.14.0-427.22.1.el9_4.x86_64   cri-o://1.31.0
```


In case of `Pending` 
```
kubectl describe pods/calico-typha-747f98cbd5-5fsbx -n calico-system
```

add to Calico custom-resources.yaml the section:

```
  controlPlaneTolerations:
    - key: node-role.kubernetes.io/control-plane
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      effect: NoSchedule
```


#### Firewall

Ensure that your hosts and firewalls allow the necessary traffic based on your configuration.
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

```
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --permanent --add-port=5473/tcp
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --permanent --add-port=51821/udp
sudo firewall-cmd --permanent --add-port=4789/udp

sudo firewall-cmd --reload
```


(For initial testing its better to disable Firewall on `control-plane` node)

```
name=K8sClusterAccept
sudo firewall-cmd --permanent --new-zone=${name}
sudo firewall-cmd --permanent --zone=${name} --set-target=ACCEPT
sudo firewall-cmd --permanent --zone=${name} --add-interface=vxlan.calico
sudo firewall-cmd --permanent --zone=${name} --add-interface="cali+"
sudo firewall-cmd --reload
```
(you can check on your local machine virtual networks created by Calico `ip a`). Example, where result is: 
```
5: calic97d13720a8@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns 2506f3b6-a7ce-45dd-b1ca-6cdf27158aca
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
8: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 66:01:39:c5:72:8a brd ff:ff:ff:ff:ff:ff
    inet 192.168.124.0/32 scope global vxlan.calico
       valid_lft forever preferred_lft forever
    inet6 fe80::6401:39ff:fec5:728a/64 scope link
       valid_lft forever preferred_lft forever
11: cali753d41897d0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns 8b60bedc-d237-4270-9a56-55715b2263f3
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
```

#### Troublesooting 

In case of multiple resets of K8s cluster, iptables could be reset
```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

In case of not starting some pod, use describe to see the issues:
```
kubectl describe pods/calico-node-wh4fk -n calico-system
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

# Kubernetes completely uninstall


```
kubectl drain <node name> — delete-local-data — force — ignore-daemonsets
kubectl delete node <node name>
```

```
kubeadm reset 
```

```
sudo yum remove kubeadm kubectl kubelet kubernetes-cni kube*
sudo yum autoremove
 
sudo rm -rf ~/.kube
```
