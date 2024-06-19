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


Possible Workaround to install CRI-O:
```
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
yum install cri-o
```


TODO: alternatives CRI-O 
- ContainerD
- Docker



3 configure yum repo for docker
```
  vim /etc/yum.repos.d/docker.repo
  [docker]
  baseurl=https://download.docker.com/linux/cen...
  gpgcheck=0
```

5 install docker for rhel8 --nobest and enable docker
```
  yum install docker-ce --nobest -y
```

6 Configuring yum repo for k8s cmds
  refer k8s docs

6 Installing kubectl kubeadm kubelet
```
  sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
7 Disable Selinux permanently
  Set SELinux in permissive mode (effectively disabling it)
  ```
  sudo setenforce 0
  #permanently disable Selinux
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

8 Permanently enabled kubelet service
```
  sudo systemctl enable --now kubelet
```

9 letting iptables see bridged traffic.

refer k8s docs -  ?? sudo modprobe br_netfilter
```
  sudo sysctl --system
```

10 docker change cgroup driver to systemd, FOR getting logs from containers
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

11 
   ```
   yum install -y iproute-tc
   ```
   useful package for vm connectivity

12 Disabling swap in ram
   ```
   vim /etc/fstab 
   #comment swap line
   ```

13 mac addess and product_uuid should be different for every node
   ```
   cat /sys/class/dmi/id/product_uuid
   ```
   the value of product_uuid and mac should be different in all nodes




# Kubernetes cluster init
```
 sudo kubeadm init --control-plane-endpoint="192.168.1.209:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
# Kubernetes cluster Reset

```
 sudo kubeadm reset
 ```


# Install Snap

Snap will be used for kubectl and HELM installation.

The EPEL repository can be added to RHEL 9 with the following command:
```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf upgrade
```
Adding the optional and extras repositories is also recommended:
```
sudo subscription-manager repos --enable "rhel-*-optional-rpms" --enable "rhel-*-extras-rpms"
sudo yum update
```

```
 sudo yum install snapd
 sudo systemctl enable --now snapd.socket
 sudo ln -s /var/lib/snapd/snap /snap

```

# Install kubectl

Snap will be used to install kubectl
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

