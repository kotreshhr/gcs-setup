# Setup Kubernetes Cluster on CentOS 7

### Prerequisites
1. **Four** machines with CentOS 7 installed. **Three** machines are sufficient
   to try but in that case master has to allow pods to be scheduled on it. These
   will be called as **kubernetes cluster nodes** going forward.
2. Setup passwordless SSH from **local machine** to all **kubernetes cluster nodes**
3. Install ansible on local machine
 ```
 On fedora/CentOS machines
#yum install ansible
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

```
3. Install kubernetes dependencies on all nodes by executing ansible playbook on **local machine**
```
#ansible-playbook -i hosts kube-dependencies.yml
```
4. Setup the master node
```
#ansible-playbook -i hosts master.yml
```
5. Check whether master node is ready
```
# kubectl get nodes
NAME                   STATUS     ROLES    AGE     VERSION
kube1.blr.redhat.com   Ready      master   1d16h   v1.13.4

```
6. Setup worker nodes
```
#ansible-playbook -i hosts workers.yml
```
7. Check whether worker nodes are ready.
> **NOTE** : It would take few minutes before worker nodes are ready
```
# kubectl get nodes
NAME                   STATUS     ROLES    AGE     VERSION
kube1.blr.redhat.com   Ready      master   1d16h   v1.13.4
kube2.blr.redhat.com   Ready      <none>   1d16h   v1.13.4
kube3.blr.redhat.com   Ready      <none>   1d16h   v1.13.4
kube4.blr.redhat.com   Ready      <none>   1d16h   v1.13.4

```
8. If **four** nodes are in kubernetes cluster. ignore this step. If **three** nodes, run below
   command to allow pods to be scheduled on master as well. This is required as gluster
   container storage expects minimum three nodes.
   
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
