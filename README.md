# Setup Gluster Container Storage (GCS) on CentOS 7 Kubernetes cluster

## Setup Kubernetes Cluster on CentOS 7
### Prerequisites
1. **Four** machines with CentOS 7 installed. **Three** machines are fine but master has to be  worker as well in that case. These will be called as **kubernetes cluster nodes** going forward.
2. Setup passwordless SSH from **local machine** to all **kubernetes cluster nodes**
3. Install ansible on local machine
 ```
 On fedora/CentOS machines
#yum install ansible
 ```
4. Disable **swap** on **kubernetes cluster nodes**
```
# blkid | grep swap
/dev/mapper/centos_benki1-swap: UUID="542afe12-ed18-49e3-b711-278e94d7ce34" TYPE="swap" 
# swapoff /dev/mapper/centos_benki1-swap
# swapoff -a

Confirm whether swap is disabled
#free -h
              total        used        free      shared  buff/cache   available
Mem:           125G        2.4G        119G        842M        4.1G        121G
Swap:            0B          0B          0B

Comment swap partition in /etc/fstab
# /etc/fstab
/dev/mapper/centos_benki1-root /                       xfs     defaults        0 0
UUID=edc44bf9-dc82-4749-bd10-13dc5290cb39 /boot                   xfs     defaults        0 0
/dev/mapper/centos_benki1-home /home                   xfs     defaults        0 0
#/dev/mapper/centos_benki1-swap swap                    swap    defaults        0 0

```
5. Disable **firewalld** on **kubernetes cluster nodes**
```
#systemctl disable firewalld
#systemctl stop firewalld
#systemctl status firewalld
```


### Steps

1. Clone this repo or download all the yml files into **local machine**
2. Edit **hosts** file to add the master and worker nodes of kubernetes cluster
```
# cat hosts
[masters]
master ansible_host=kube1.lab.eng.blr.redhat.com ansible_user=root

[workers]
worker1 ansible_host=kube2.blr.redhat.com ansible_user=root
worker2 ansible_host=kube3.blr.redhat.com ansible_user=root
worker3 ansible_host=kube4.blr.redhat.com ansible_user=root
[root@benki1 kube-cluster]# 

```
3.  Install kubernetes dependencies on all nodes by executing ansible playbook on **local machine**
```
#ansible-playbook -i hosts kube-dependencies.yml
```
4. Setup the cgroup driver of docker to **systemd**  on all **kubernetes cluster nodes**
> **NOTE**: kubelet cgroup driver and docker cgroup driver should be same. It should be **systemd** and not **cgroupfs** as cgroupfs is unstable at large workloads.  kubelet cgroup driver is **systemd** by default
Check this [link](https://kubernetes.io/docs/setup/cri/) for more details.  

```  
## Create /etc/docker directory.  
mkdir /etc/docker  
  
# Setup daemon.  
cat > /etc/docker/daemon.json <<EOF  
{  
  "exec-opts": ["native.cgroupdriver=systemd"],  
  "log-driver": "json-file",  
  "log-opts": {  
"max-size": "100m"  
  },  
  "storage-driver": "overlay2",  
  "storage-opts": [  
"overlay2.override_kernel_check=true"  
  ]  
}  
EOF  
  
mkdir -p /etc/systemd/system/docker.service.d  
  
# Restart docker.  
systemctl daemon-reload  
systemctl restart docker  
```
5. Setup the master node
```
#ansible-playbook -i hosts master.yml
```
6. Check whether master node is ready
```
# kubectl get nodes
NAME                   STATUS     ROLES    AGE     VERSION
kube1.blr.redhat.com   Ready      master   1d16h   v1.13.4

```
7. Setup worker nodes
```
#ansible-playbook -i hosts workers.yml
```
8. Check whether worker nodes are ready.
> **NOTE** : It would take few minutes before worker nodes are ready
```
# kubectl get nodes
NAME                   STATUS     ROLES    AGE     VERSION
kube1.blr.redhat.com   Ready      master   1d16h   v1.13.4
kube2.blr.redhat.com   Ready      <none>   1d16h   v1.13.4
kube3.blr.redhat.com   Ready      <none>   1d16h   v1.13.4
kube4.blr.redhat.com   Ready      <none>   1d16h   v1.13.4

```

## Deploy Gluster Container Storage (GCS)
1. Install kubectl gluster tool
```
#sudo pip install kubectl-gluster
```
2. Prepare **gcs yaml** file
```
#cat gcs.yml
[root@kube1]# cat ~/gcs.yaml 
namespace: gcs
cluster-size: 3
nodes:
    - address: kube2.blr.redhat.com
      devices: ["/dev/sdb", "/dev/sdc", "/dev/sdd"]

    - address: kube3.blr.redhat.com
      devices: ["/dev/sdb", "/dev/sdc", "/dev/sdd"]

    - address: kube4.blr.redhat.com
      devices: ["/dev/sdb", "/dev/sdc", "/dev/sdd"]

```
where
```
namespace    - Cluster namespace, useful when managing multiple
                   clusters
cluster-size - Number of nodes in Gluster cluster. Currently only 3
                   nodes are supported.
nodes        - Details of nodes where Gluster server pods needs to be
                   deployed.
address      - Address of node as listed in kubectl get nodes(Use
                   `kubectl get nodes` to get the address)
devices      - Raw devices which are required to auto provision
                   Gluster bricks during Volume create
```
3. Deploy **GCS** from **master** node
```
#kubectl gluster deploy ~/gcs.yml
```
