---
title: Using Ceph RBD for persistent storage
date: 2019-09-16 22:34:02
categories: DevOps
---
## Overview

This topic provides a complete example of using an existing Ceph cluster for OKD persistent storage. It is assumed that a working Ceph cluster is already set up. If not, consult the [Overview of Red Hat Ceph Storage](https://access.redhat.com/products/red-hat-ceph-storage).

Persistent Storage Using Ceph Rados Block Device provides an explanation of persistent volumes (PVs), persistent volume claims (PVCs), and using Ceph RBD as persistent storage.


## Using an existing Ceph cluster for persistent storage

<!-- more -->

* Install the ceph-common package same as ceph-cluster.

```yaml
    - name: update apt cache
      apt:
         update_cache=yes

    - name: install ceph-common
      apt:
        name: ceph-common
      state: present
```

`NOTE`: The **ceph-common** library must be installed on **all nodes**.

* Create the keyring for the client: **(On Ceph Node)**

```zsh
$ ceph auth get-or-create client.qemu mon 'allow *' osd 'allow rwx pool=rbd' -o /etc/ceph/ceph.client.qemu.keyring
```

* Convert the keyring to base64: **(On Ceph Node)**

```zsh
$ grep key /etc/ceph/ceph.client.qemu.keyring | awk '{printf "%s", $NF}' | base64
```

`NOTE`: This base64 key is generated on one of the Ceph MON nodes using the **ceph auth get-key client.admin | base64** command, then copying the output and pasting it as the secret key’s value.


## Create StorageClass(`Dynamic Volume Provisioning`) using Ceph RBD

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elk
---
# define admin secret on cluster level, create PV
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-admin
  namespace: kube-system
data:
  key: QVFEQUNWWmRmTUxBSkJBQXlSV88rUm11RzJSb0J2Tk9SVllSaGc9PQ==
---
# define user secret on namespace level, create PVC
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-qemu
  namespace: elk
data:
  key: QVFCNldYTmRtZ29iREJBQSt4dXorNHp0Wi33RWluR1J4U1hWcnc9PQ==
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 192.168.0.2:6789,192.168.0.3:6789
  pool: rbd
  adminId: admin
  adminSecretNamespace: kube-system
  adminSecretName: ceph-secret-admin
  userId: qemu
  userSecretName: ceph-secret-qemu
  userSecretNamespace: elk
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
---
# StatefulSet
......
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
      storageClassName: ceph-rbd    
```


**[bug & fix](
https://github.com/kubernetes/kubernetes/issues/38923#issuecomment-315255075)**:

```text
Warning ProvisioningFailed 31s (x16 over 19m) persistentvolume-controller Failed to provision volume with StorageClass "ceph-elk": failed to create rbd image: executable file not found in $PATH, command output:
```

Please check that `rbac/deployment.yaml` **quay.io/external_storage/rbd-provisioner:latest** image has the same Ceph version installed as your Ceph cluster. You can check it like this on any machine running docker:

```zsh
ceph-cluster:~$ ceph version
ceph version 12.2.12 (1436006594665279fe734b4c15d7e08c13ebd777) luminous (stable)

$ docker history quay.io/external_storage/rbd-provisioner:v1.0.0-k8s1.10 | grep CEPH_VERSION
<missing>           15 months ago       /bin/sh -c #(nop)  ENV CEPH_VERSION=luminous    0B
```

## Create Static PV using Ceph RBD

```zsh
ceph-cluster:~$ rbd create vmd0 -s 64G
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elk
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-qemu
  namespace: elk
data:
  key: QVFCNldYTmRtZ29iREJBQSt4dXorNHp0Wi33RWluR1J4U1hWcnc9PQ==
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-elk-pv0
  namespace: elk
spec:
  capacity:
    storage: 64Gi
  # The 'accessModes' are used as labels to match a PV and a PVC. They currently do not define any form of access control. All block storage is defined to be single user (non-
shared storage).
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    - 192.168.0.2:6789
    - 192.168.0.3:6789
    pool: rbd
    # The 'vmd0' must be created on the Ceph cluster.
    image: vmd0
    user: admin
    secretRef:
      name: ceph-secret-qemu
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-elk-claim0
  namespace: elk
spec:
  # The 'accessModes' do not enforce access rights but instead act as labels to match a PV to a PVC.
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 64Gi
---
# pod
......
    volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ceph-elk-claim0      
```
