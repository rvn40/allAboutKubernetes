# Setup Kubernetes Cluster using Kubeadm on CentOS 7

Deploy 2 virtual machines with centos 7 to deploy a kubernetes cluster.
The scenario is 1 vm would be setting up as the master and the other one is the worker.

## Spesifications

|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|master01.example.com|172.42.42.100|CentOS 7|2G|2|
|Worker|worker01.example.com|172.42.42.101|CentOS 7|1G|1|

### Pre-requisites
##### Install, enable and start docker service
Use the bash script below to install docker. In this case, I use docker ce version "18.03.0.ce-1.el7.centos". Please, feel free to use any version as long as that version is compatible with kubernetes. This script is only able to run on centos.
```
#!/bin/sh


DEFAULT_DOWNLOAD_URL="https://download.docker.com"
if [ -z "$DOWNLOAD_URL" ]; then
        DOWNLOAD_URL=$DEFAULT_DOWNLOAD_URL
fi

DEFAULT_REPO_FILE="docker-ce.repo"
if [ -z "$REPO_FILE" ]; then
        REPO_FILE="$DEFAULT_REPO_FILE"
fi


get_distribution() {
        lsb_dist=""
        # Every system that we officially support has /etc/os-release
        if [ -r /etc/os-release ]; then
                lsb_dist="$(. /etc/os-release && echo "$ID")"
        fi
        # Returning an empty string here should be alright since the
        # case statements don't act unless you provide an actual value
        echo "$lsb_dist"
}

do_install() {

        if type "docker" > /dev/null; then

                echo "We found your machine has already installed by docker. If you willing to continue this process press (y) and we would uninstall the existing package"
                read  -n 1 -p "Do you want to continue? " uninstall

                if [ "$uninstall" != "y" ]; then
                        exit 1
                fi
        fi

        echo "=============== Preparing Installation ==============="

        lsb_dist=$( get_distribution )
        lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"

        case "$lsb_dist" in
                centos)
                        if [ -z "$dist_version" ] && [ -r /etc/os-release ]; then
                                dist_version="$(. /etc/os-release && echo "$VERSION_ID")"
                                echo $dist_version
                        fi
        ;;
        esac

        case "$lsb_dist" in
                centos|fedora)
                        yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

                        yum update -y

                        yum install -y yum-utils curl

                        yum_repo="$DOWNLOAD_URL/linux/$lsb_dist/$REPO_FILE"
                        if ! curl -Ifs "$yum_repo" > /dev/null; then
                                echo "Error: Unable to curl repository file $yum_repo, is it valid?"
                                exit 1
                        fi

                        yum-config-manager --add-repo $yum_repo

                        yum list docker-ce --showduplicates | sort -r

                        read -p "Choose the version you need: " dv

                        yum install docker-ce-$dv -y

                ;;
        esac

}

do_install
```

##### Disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```
##### Disable Firewall
```
systemctl disable firewalld
systemctl stop firewalld
```
##### Disable swap
```
sed -i '/swap/d' /etc/fstab
swapoff -a
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
### Kubernetes Setup
##### Add yum repository
```
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
##### Install Kubernetes
```
yum install -y kubeadm kubelet kubectl
```
##### Enable and Start kubelet service
```
systemctl enable kubelet
systemctl start kubelet
```
## On kmaster
##### Initialize Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=172.42.42.100 --pod-network-cidr=192.168.0.0/16
```
##### Copy kube config
To be able to use kubectl command to connect and interact with the cluster, the user needs kube config file.

In my case, the user account is venkatn
```
mkdir /home/venkatn/.kube
cp /etc/kubernetes/admin.conf /home/venkatn/.kube/config
chown -R venkatn:venkatn /home/venkatn/.kube
```
##### Deploy Calico network
This has to be done as the user in the above step (in my case it is __venkatn__)
```
kubectl create -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

##### Cluster join command
```
kubeadm token create --print-join-command
```
## On Kworker
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```
