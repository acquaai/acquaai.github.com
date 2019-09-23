---
title: GlusterFS Native Storage Service for Kubernetes
date: 2019-09-23 21:11:38
categories: DevOps
---
## Configuring GlusterFS

```zsh
$ git clone https://github.com/gluster/gluster-kubernetes.git
```

Copy the `deploy/` directory to the master node of the Kubernetes cluster.

You will have to provide your own topology file. A sample topology file is included in the `deploy/` directory (default location that gk-deploy expects) which can be used as the topology for the vagrant libvirt setup. When creating your own topology file:

- Make sure the topology file only lists block devices intended for heketi's use. heketi needs access to whole block devices (e.g. /dev/sdb, /dev/vdb) which it will partition and format.
- The `hostnames` array is a bit misleading. `manage` should be a list of hostnames for the node, but `storage` should be a list of IP addresses on the node for backend storage communications.

<!-- more -->

```zsh
$ cat topology.json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "w-192-168-16-14"
              ],
              "storage": [
                "192.168.16.14"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        ......
      ]
    }
  ]
}
```

**Installing gluster client on related worker nodes**

```zsh
sudo apt-get install -y glusterfs-client
```

## Deploy heketi and GlusterFS

Verify the Kubernetes installation by making sure all nodes are Ready:

```zsh
$ kubectl get cs,nodes
```

```zsh
$ ./gk-deploy -g --admin-key <key> --user-key <key>
```

`NOTE:`

```tex
Welcome to the deployment tool for GlusterFS on Kubernetes and OpenShift.

Before getting started, this script has some requirements of the execution
environment and of the container platform that you should verify.

The client machine that will run this script must have:
 * Administrative access to an existing Kubernetes or OpenShift cluster
 * Access to a python interpreter 'python'

Each of the nodes that will host GlusterFS must also have appropriate firewall
rules for the required GlusterFS ports:
 * 2222  - sshd (if running GlusterFS in a pod)
 * 24007 - GlusterFS Management
 * 24008 - GlusterFS RDMA
 * 49152 to 49251 - Each brick for every volume on the host requires its own
   port. For every new brick, one new port will be used starting at 49152. We
   recommend a default range of 49152-49251 on each host, though you can adjust
   this to fit your needs.

The following kernel modules must be loaded:
 * dm_snapshot
 * dm_mirror
 * dm_thin_pool

For systems with SELinux, the following settings need to be considered:
 * virt_sandbox_use_fusefs should be enabled on each node to allow writing to
   remote GlusterFS volumes

In addition, for an OpenShift deployment you must:
 * Have 'cluster_admin' role on the administrative account doing the deployment
 * Add the 'default' and 'router' Service Accounts to the 'privileged' SCC
 * Have a router deployed that is configured to allow apps to access services
   running in the cluster

Do you wish to proceed with deployment?

[Y]es, [N]o? [Default: Y]: 
```

A few minutes later, you may will get some resources like this:

```zsh
$ kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
glusterfs-gmx44           1/1     Running   0          31m
glusterfs-gzpvg           1/1     Running   0          31m
glusterfs-rb7t7           1/1     Running   0          31m
heketi-8658674597-bztc8   1/1     Running   0          16m

$ kubectl get endpoints
NAME                       ENDPOINTS                                                  AGE
heketi                     10.244.84.58:8080                                          17m
heketi-storage-endpoints   192.168.16.14:1,192.168.16.15:1,192.168.16.16:1            16m
kubernetes                 192.168.16.11:6443,192.168.16.12:6443,192.168.16.13:6443   31d

$ kubectl get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
heketi                     ClusterIP   10.100.236.131   <none>        8080/TCP   17m
heketi-storage-endpoints   ClusterIP   10.109.192.23    <none>        1/TCP      18m
kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP    31d

$ export HEKETI_CLI_SERVER=$(kubectl get svc/heketi --template 'http://{{.spec.clusterIP}}:{{(index .spec.ports 0).port}}')

$ echo $HEKETI_CLI_SERVER
http://10.100.236.131:8080

$ kubectl create -f gluster-storage-class.yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: gluster-heketi
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.100.236.131:8080"
  restuser: "admin"
  restuserkey: "<--key-->"
  
$ kubectl create -f gluster-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: gluster1
 annotations:
   volume.beta.kubernetes.io/storage-class: gluster-heketi
spec:
 accessModes:
  - ReadWriteOnce
 resources:
   requests:
     storage: 5Gi
     
$ kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
gluster1   Bound    pvc-012cdd87-0fe6-4dcc-9279-f07c853069f7   5Gi        RWO            gluster-heketi   11s

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS     REASON   AGE
pvc-012cdd87-0fe6-4dcc-9279-f07c853069f7   5Gi        RWO            Delete           Bound    default/gluster1   gluster-heketi            39s     
```


**Question**:

```yaml
......
Controlled By:  Job/heketi-storage-copy-job
Containers:
  heketi:
    Container ID:  
    Image:         heketi/heketi:dev # where to modify?
......    
```


**Ref**

+ [Gluster-Kubernetes](https://github.com/gluster/gluster-kubernetes)
+ [GlusterFS Dynamic Provisioning](https://github.com/gluster/gluster-kubernetes/blob/master/docs/examples/hello_world/README.md)
