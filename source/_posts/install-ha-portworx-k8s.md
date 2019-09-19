---
title: Install Portworx Cluster on Kubernetes(on-premise)
date: 2019-09-19 22:22:13
catagory: DevOps
---
## Prepare hosts with storage

Portworx (PX) requires at least some nodes in the cluster to have dedicated storage for Portworx to use. PX will then carve out virtual volumes from these storage pools. In this example, we use a 3.3T block device that exists on each node.

List block devices on worker nodes

```zsh
$ lsblk
NAME                              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                                 8:0    0  1.1T  0 disk 
├─sda1                              8:1    0  512M  0 part /boot/efi
└─sda2                              8:2    0  1.1T  0 part 
  ├─w--192--31--16--16--vg-root   253:0    0  1.1T  0 lvm  /
  └─w--192--31--16--16--vg-swap_1 253:1    0  976M  0 lvm  
sdb                                 8:16   0  3.3T  0 disk
```

Note the storage device sdb, which will be used by PX as one of it's raw block disks. All the nodes in this setup have the sdb device.

<!-- more -->

## Generate the specs

+ **[Generating the Portworx specs](https://docs.portworx.com/portworx-install-with-kubernetes/on-premise/other/#)**
 + **[My spec](https://github.com/acquaai/Kubernetes/blob/master/CSI/Portworx/spec.yaml)**

Get Kubernetes Version:

```zsh
kubectl version --short | awk -Fv '/Server Version: / {print $3}'
```

Portworx will create and manage an internal key-value store (kvdb) cluster.

You can restrict the nodes that will run the key-value store by labelling your nodes with the label px/metadata-node=true. Only the nodes with the label will participate in the kvdb cluster. This allows you to use nodes with dedicated hardware for the key-value store.

**For example**:

```zsh
kubectl label nodes node1 node2 node3 px/metadata-node=true
```


## Apply the specs

```zsh
kubectl create secret generic registrykey --from-file=.dockerconfigjson=.docker/config.json --type=kubernetes.io/dockerconfigjson -n kube-system
kubectl apply -f spec.yaml
```

`NOTE`: **portworx/px-enterprise:2.1.5** image is need to  be.


## Monitor the portworx pods

```zsh
kubectl get pods -o wide -n kube-system -l name=portworx
```


## Monitor Portworx cluster status

```zsh
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
```

```text
Status: PX is operational
License: Trial (expires in 31 days)
Node ID: 4078b64e-2e1c-4865-aeab-2da8afa86d43
        IP: 192.31.16.16 
        Local Storage Pool: 1 pool
        POOL    IO_PRIORITY     RAID_LEVEL      USABLE  USED    STATUS  ZONE    REGION
        0       LOW             raid0           3.3 TiB 10 GiB  Online  default default
        Local Storage Devices: 1 device
        Device  Path            Media Type              Size            Last-Scan
        0:1     /dev/sdb2       STORAGE_MEDIUM_MAGNETIC 3.3 TiB         19 Sep 19 09:07 UTC
        total                   -                       3.3 TiB
        Journal Device: 
        1       /dev/sdb1       STORAGE_MEDIUM_MAGNETIC
Cluster Summary
        Cluster ID: px-cluster-d620cbe1-ad1c-4b44-be2e-043929a7080f
        Cluster UUID: c50eb3ec-d4ba-416a-8a66-643efb0cb312
        Scheduler: kubernetes
        Nodes: 3 node(s) with storage (3 online)
        IP              ID                                      SchedulerNodeName       StorageNode     Used    Capacity        Status  StorageStatus   Version         Kernel        OS
        192.31.16.14    bb755d05-2c28-4a57-9126-4032ae29ad56    w-192-31-16-14          Yes             10 GiB  3.3 TiB         Online  Up              2.1.5.0-3b73452 4.15.0-58-generic      Ubuntu 18.04.3 LTS
        192.31.16.16    4078b64e-2e1c-4865-aeab-2da8afa86d43    w-192-31-16-16          Yes             10 GiB  3.3 TiB         Online  Up (This node)  2.1.5.0-3b73452 4.15.0-58-generic      Ubuntu 18.04.3 LTS
        192.31.16.15    00b67c6d-d4c1-4eab-9a71-fe07040fd98a    w-192-31-16-15          Yes             10 GiB  3.3 TiB         Online  Up              2.1.5.0-3b73452 4.15.0-58-generic      Ubuntu 18.04.3 LTS
Global Storage Pool
        Total Used      :  31 GiB
        Total Capacity  :  9.8 TiB
```


+ [Portworx Licensing](https://docs.portworx.com/reference/knowledge-base/px-licensing/)
 + [PX-Developer](https://github.com/portworx/px-dev)
