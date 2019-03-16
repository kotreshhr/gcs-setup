# Setup Gluster Container Storage (GCS) on CentOS 7 Kubernetes cluster

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
