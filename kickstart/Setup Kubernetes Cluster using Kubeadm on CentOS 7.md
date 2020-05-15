# Setup Kubernetes Cluster using Kubeadm on CentOS 7

Deploy 2 virtual machines with centos 7 to deploy a kubernetes cluster.
The scenario is 1 vm would be setting up as the master and the other one is the worker.

## Spesifications

|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|master01.example.com|172.42.42.100|CentOS 7|2G|2|
|Worker|worker01.example.com|172.42.42.101|CentOS 7|1G|1|

