---
title: K8S持久化卷
date: 2018-03-10 11:40:44
categories: DevOps
---
> 声明：本文来自[jimmysong](https://jimmysong.io/posts/kubernetes-persistent-volume/)的文章，整理记录在此仅为加深对 K8S PV 学习和理解。

在 K8S 中没有使用**持久化卷**是不完整的，PV作为子系统为管理员和用户提供了API。
 
## APIs
 
`PersistentVolume`（PV）管理员配置的存储，作为 K8S 群集的一部分。类似节点是集群中的资源一样，PV 也是集群中的资源。 PV 是 Volume 之类的卷插件，具有独立于 使用PV的Pod 的生命周期。卷的生命比 Pod 中的所有容器都长，当容器重启时数据仍然得以保存。只有当 Pod 消亡时，卷将不复存在。Kubernetes 支持多种类型的卷，Pod 可以同时使用任意数量的卷。该API对象包含存储实现的细节，即 NFS、RBD 或云厂商的存储系统。

`PersistentVolumeClaim`（PVC）用户对存储的请求，与 Pod 类似。Pod 消耗节点资源，PVC 消耗 PV 资源。PVC 请求特定的大小和访问模式。

## 卷和声明(claim)的生命周期

PV作为集群中的一项资源，PVC消费PV资源，也作为对资源的消费检查。

<!-- more -->

### 配置（Provision）

#### 静态 PV

集群管理员创建的PV，供集群用户使用的实际存储细节，用于消费。

#### 动态 PV

当管理员创建的静态PV不匹配用户的消费时，集群可能会尝试动态地为 PVC 创建卷。此配置基于 StorageClasses：PVC 必须请求存储类，且管理员必须创建并配置该类才能进行动态创建。声明该类为 "" 可以有效地禁用其动态配置。

要启用基于存储级别的动态存储配置，集群管理员需要启用 API server 上的 DefaultStorageClass 准入控制器。例如，通过确保 DefaultStorageClass 位于 API server 组件的 --admission-control 标志，使用逗号分隔的有序值列表中，可以完成此操作。

### 绑定

在动态配置的情况下，用户创建或已经创建了具有特定存储量的 PersistentVolumeClaim 以及某些访问模式。master 中的控制环路监视新的 PVC，寻找匹配的 PV（如果可能），并将它们绑定在一起。如果为新的 PVC 动态调配 PV，则该环路始终将该 PV 绑定到 PVC。否则，用户总会得到他们所请求的存储，但是容量可能超出要求的数量。一旦 PV 和 PVC 绑定后，PersistentVolumeClaim 绑定是排它性的，不管它们是如何绑定的。 PVC 跟 PV 绑定是一对一的映射。

如果没有匹配的卷，声明将无限期地保持未绑定状态。随着匹配卷的可用，声明将被绑定。例如，配置了许多 50Gi PV的集群将不会匹配请求 100Gi 的PVC。将100Gi PV 添加到群集时，可以绑定 PVC。

### 使用

用户进行了声明，并且该声明是绑定的，则只要用户需要，绑定的 PV 就属于该用户。用户通过在 Pod 的 volume 配置中包含 persistentVolumeClaim 来调度 Pod 并访问用户声明的 PV。

### 持久化卷声明的保护

PVC 保护的目的是确保由 pod 正在使用的 PVC 不会从系统中移除，避免数据丢失或损坏。

`当 pod 状态为 Pending 并且 pod 已经分配给节点或 pod 为 Running 状态时，PVC 处于活动状态。`

+ A v1.9 or higher Kubernetes must be installed.
+ As PVC Protection is a Kubernetes v1.9 alpha feature it must be enabled:
 1. Admission controller must be started with the PVC Protection plugin.
 2. All Kubernetes components must be started with the PVCProtection alpha features enabled.

### 回收

用户用完 volume 后，可以从允许回收资源的 API 中删除 PVC 对象。PersistentVolume 的回收策略告诉集群在存储卷声明释放后应如何处理该卷：`保留 | 回收 | 删除`

#### 保留

允许手动回收资源。当 PersistentVolumeClaim 被删除时，PersistentVolume 仍然存在，volume 被视为“已释放”。

由于前一个声明人的数据仍然存在，所以还不能马上进行其他声明。管理员可以通过以下步骤手动回收卷：
1. 删除 PersistentVolume。在删除 PV 后，外部基础架构中的关联存储资产（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）仍然存在。
2. 手动清理相关存储资产上的数据。
3. 手动删除关联的存储资产，或者如果要重新使用相同的存储资产，再使用存储资产定义创建新的 PersistentVolume。

#### 删除

对于支持删除回收策略的卷插件，删除操作将从 Kubernetes 中删除 PersistentVolume 对象，并删除外部基础架构（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中的关联存储资产。动态配置的卷继承其 StorageClass 的回收策略，默认为 Delete。管理员应该根据用户的期望来配置 StorageClass，否则就必须要在 PV 创建后进行编辑或修补。[persistentVolumeReclaimPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#persistentvolumeclaim-v1-core)

#### 回收

如果存储卷插件支持，回收策略会在 volume 上执行基本擦除（rm -rf / thevolume / *），可被再次声明使用。

管理员可以使用 [Kubernetes controller manager 命令行参数](https://kubernetes.io/docs/reference/generated/kube-controller-manager/)来配置自定义回收站 pod 模板。自定义回收站 pod 模板必须包含 volumes 规范，volumes 部分的自定义回收站模块中指定的特定路径将被替换为正在回收的卷的特定路径。

### 扩展持久化卷声明

K8S v1.9版本中，支持扩展持久化卷类型：

+ gcePersistentDisk
+ awsElasticBlockStore
+ Cinder
+ glusterfs
+ rbd

管理员可以通过设置 ExpandPersistentVolumes 特性为 true来 允许扩展持久卷声明。管理员还应该启用[PersistentVolumeClaimResize](https://kubernetes.io/docs/admin/admission-controllers/#persistentvolumeclaimresize)准入控制插件来调整大小的卷的其他验证。

只要 PersistentVolumeClaimResize 准入插件开启，将只允许 allowVolumeExpansion 字段设置为 true 的存储类进行大小调整。

在任何情况下都不会创建新的 PersistentVolume 来满足声明。 Kubernetes 将尝试调整现有 volume 来满足声明的要求。

扩展卷包括 文件系统，只有在`ReadWrite`模式下使用 PersistentVolumeClaim 启动新Pod时，才会执行文件系统大小调整。 换句话说，如果正在扩展的卷在pod或部署中使用，则需要删除并重新创建pod。只有`XFS, Ext3, Ext4`支持文件系统大小调整。

## 持久化卷类型

PersistentVolume以插件的形式实现。Kubernetes目前支持的卷类型:

+ GCEPersistentDisk
+ AWSElasticBlockStore
+ AzureFile
+ AzureDisk
+ FC (Fibre Channel)**
+ FlexVolume
+ Flocker
+ NFS
+ iSCSI
+ RBD (Ceph Block Device)
+ CephFS
+ Cinder (OpenStack block storage)
+ Glusterfs
+ VsphereVolume
+ Quobyte Volumes
+ HostPath (Single node testing only – local storage is not supported in any way and WILL NOT WORK in a multi-node cluster)
+ VMware Photon
+ Portworx Volumes
+ ScaleIO Volumes
+ StorageOS
+ Raw Block Support exists for these plugins only.

### 持久化卷类型示例

#### emptyDir

当 Pod 被分配给节点时，首先创建 emptyDir 卷，只要该 Pod 在该节点上运行，该卷就会存在，但它最初是空的。Pod 中的容器可以读取和写入 emptyDir 卷中的相同文件，尽管该卷可以挂载到每个容器中的相同或不同路径上。当从节点中删除 Pod 时，emptyDir 中的数据将被永久删除。

容器崩溃不会从节点中移除 pod，因此 emptyDir 卷中的数据在容器崩溃时是安全的。

emptyDir 的用法：

+ 暂存空间，例如用于基于磁盘的合并排序
+ 用作长时间计算崩溃恢复时的检查点
+ Web服务器容器提供数据时，保存内容管理器容器提取的文件

example:

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

#### hostPath

hostPath 卷将主机节点的文件系统中的文件或目录挂载到集群中。该功能大多数 Pod 都用不到(跨节点访问)。

hostPath 的用途如下：

+ 运行需要访问 Docker 内部的容器；使用 /var/lib/docker 的 hostPath
+ 在容器中运行 cAdvisor；使用 /dev/cgroups 的 hostPath
+ 允许 pod 指定给定的 hostPath 是否应该在 pod 运行之前存在，是否应该创建，以及它应该以什么形式存在

注意：
+ 由于每个节点上的文件都不同，具有相同配置（例如从 podTemplate 创建的）的 pod 在不同节点上的行为可能会有所不同
+ 当 Kubernetes 按照计划添加资源感知调度时，将无法考虑 hostPath 使用的资源
+ 在底层主机上创建的文件或目录只能由 root 写入。您需要在特权容器中以 root 身份运行进程，或修改主机上的文件权限以便写入 hostPath 卷

example:

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

#### cephfs

cephfs 卷允许将现有的 CephFS 卷挂载到容器中，与 emptyDir 不同的是，当删除 Pod 时cephfs 卷上的内容被保留，仅仅是卸载卷。与 awsElasticBlockStore 一样可以预先填充数据，并且可以在数据包之间"切换"数据。 CephFS 能被多个写设备同时挂载。

[CephFS example](https://github.com/kubernetes/examples/tree/master/staging/volumes/cephfs/)

#### secret

secret 卷用于将敏感信息（如密码）传递到 pod。可以将 secret 存储在 Kubernetes API 中，并将它们挂载为文件，以供 Pod 使用，而无需直接连接到 Kubernetes。 secret 卷由 tmpfs（一个 RAM 支持的文件系统）支持，永远不会写入非易失性存储器。必须先在 Kubernetes API 中创建一个 secret，然后才能使用它。

[Secrets are described in more detail here.](https://kubernetes.io/docs/concepts/configuration/secret/)
[secret example](https://jimmysong.io/posts/kubernetes-secret-configuration/)

#### nfs

nfs 卷允许 NFS（网络文件系统）共享挂载到容器中，与 emptyDir 不同的是，当删除 Pod 时 nfs 卷上的内容被保留，仅仅是卸载卷。与 cephfs 一样可以预先填充数据，并且可以在数据包之间"切换"数据。 nfs 能被多个写设备同时挂载。

[NFS example](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs)

#### projected

[projected example](https://kubernetes.io/docs/concepts/storage/volumes/#projected)

#### rbd

rbd 卷允许将 Rados Block Device 卷挂载到容器中。与 emptyDir 不同的是，当删除 Pod 时 nfs 卷上的内容被保留，仅仅是卸载卷。与 cephfs 一样可以预先填充数据，并且可以在数据包之间"切换"数据。可以同时为多个用户以**`只读`**方式挂载，由单个用户以读写模式安装——不允许同时写入。

[RBD example](https://github.com/kubernetes/examples/tree/master/staging/volumes/rbd)

## 持久化卷

每个 PV 配置中都包含一个 sepc 规格字段和一个 status 卷状态字段。

### 容量

使用 PV 的容量属性设置存储容量。Kubernetes 资源模型[capacity](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/resources.md)

### 卷模式

v1.9 之前，所有卷插件默认在持久卷上创建一个文件系统。在 v1.9 中，除文件系统之外，开始支持块设备。用户现在可以指定一个 volumeMode， volumeMode 的有效值可以是“Filesystem”或“Block”。如果未指定，volumeMode 将默认为“Filesystem”。

### 访问模式

PersistentVolume 以卷资源提供者支持的模式挂载到主机上，如下：

+ （RWO）ReadWriteOnce——该卷可以被单个节点以读/写模式挂载
+ （ROX）ReadOnlyMany——该卷可以被多个节点以只读模式挂载
+ （RWX）ReadWriteMany——该卷可以被多个节点以读/写模式挂载

> **!!!** 一个卷一次只能使用一种访问模式挂载，即使它支持很多访问模式。例如，GCEPersistentDisk 可以由单个节点作为 ReadWriteOnce 模式挂载，或由多个节点以 ReadOnlyMany 模式挂载，但不能同时挂载。

卷资源提供者支持的访问模式：

| Volume Plugin | ReadWriteOnce | ReadOnlyMany | ReadWriteMany |
| :------------ | :-----------: | :-----------:| :------------:|
| AWSElasticBlockStore | ✓ | - | - |
| AzureFile | ✓ | ✓ | ✓ |
| AzureDisk | ✓ | - | - |
| CephFS    | ✓ | ✓ | ✓ |
| Cinder    | ✓ | - | - |
| FC        | ✓ | ✓ | - |
| FlexVolume| ✓ | ✓ | - |
| Flocker   | ✓ | - | - |
| GCEPersistentDisk | ✓ | ✓ | - |
| Glusterfs | ✓ | ✓ | ✓ |
| HostPath  | ✓ | - | - |
| iSCSI     | ✓ | ✓ | - |
| PhotonPersistentDisk | ✓ | - | - |
| Quobyte   | ✓ | ✓ | ✓ |
| NFS       | ✓ | ✓ | ✓ |
| RBD       | ✓ | ✓ | - |
| VsphereVolume | ✓ | - | - (works when pods are collocated)|
| PortworxVolume| ✓ | - | ✓ |
| ScaleIO   | ✓ | ✓ | - |
| StorageOS | ✓ | - | - |

### 类

PV 可以具有一个类，通过将 storageClassName 属性设置为 [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) 的名称来指定该类。一个特定类别的 PV 只能绑定到请求该类别的 PVC。没有 storageClassName 的 PV 就没有类，它只能绑定到不需要特定类的 PVC。

### 回收策略

当前的回收策略包括：

+ Retain（保留）——手动回收
+ Recycle（回收）——基本擦除（rm -rf /thevolume/*）
+ Delete（删除）——关联的存储资产（例如 AWS EBS、GCE PD、Azure Disk 和 OpenStack Cinder 卷）将被删除

当前只有 NFS 和 HostPath 支持回收策略。AWS EBS、GCE PD、Azure Disk 和 Cinder 卷支持删除策略。

### 挂载选项

Kubernetes 管理员可以指定在节点上为挂载持久卷指定挂载选项，**但不是所有的**持久化卷类型都支持挂载选项。挂载选项没有校验，如果挂载选项无效则挂载失败。

### 状态

命令行会显示绑定到 PV 的 PVC 的名称。卷可以处于以下的某种状态：

+ Available（可用）——一块空闲资源还没有被任何声明绑定
+ Bound（已绑定）——卷已经被声明绑定
+ Released（已释放）——声明被删除，但是资源还未被集群重新声明
+ Failed（失败）——该卷的自动回收失败

## PersistentVolumeClaim

每个 PVC 中同样包含一个 sepc 规格字段和一个 status 声明状态字段。

### 访问模式

在请求具有特定访问模式的存储时，声明使用与卷相同的约定。

### 卷模式

声明使用与卷相同的约定，指示将卷作为文件系统或块设备使用。

### 资源

像 pod 一样，声明可以请求特定数量的资源。在这种情况下，请求是用于存储的。相同的资源模型适用于卷和声明。

### 选择器

声明可以指定一个标签选择器来进一步过滤该组卷。只有标签与选择器匹配的卷可以绑定到声明。选择器由两个字段组成：

+ matchLabels：volume 必须有具有该值的标签
+ matchExpressions：这是一个要求列表，通过指定关键字，值列表以及与关键字和值相关的运算符组成。有效的运算符包括 In、NotIn、Exists 和 DoesNotExist。

所有来自 matchLabels 和 matchExpressions 的要求都被"与"（and）——它们必须全部满足才能匹配。

### 类

声明可以通过使用属性 storageClassName 指定 StorageClass 的名称来请求特定的类。只有所请求的类与 PVC 具有相同 storageClassName 的 PV 才能绑定到 PVC。

PVC 不一定要请求类。其 storageClassName 设置为 "" 的 PVC 始终被解释为没有请求类的 PV，因此只能绑定到没有类的 PV（没有注解或 ""）。没有 storageClassName 的 PVC 根据是否打开DefaultStorageClass 准入控制插件，集群对其进行不同处理。

+ 如果打开了准入控制插件，管理员可以指定一个默认的 StorageClass。所有没有 StorageClassName 的 PVC 将被绑定到该默认的 PV。通过在 StorageClass 对象中将注解 storageclassclass.ubernetes.io/is-default-class 设置为 “true” 来指定默认的 StorageClass。如果管理员没有指定缺省值，那么集群会响应 PVC 创建，就好像关闭了准入控制插件一样。如果指定了多个默认值，则准入控制插件将禁止所有 PVC 创建。
+ 如果准入控制插件被关闭，则没有默认 StorageClass 的概念。所有没有 storageClassName 的 PVC 只能绑定到没有类的 PV。在这种情况下，没有 storageClassName 的 PVC 的处理方式与 storageClassName 设置为 "" 的 PVC 的处理方式相同。

当 PVC 指定了 selector，除了请求一个 StorageClass 之外，这些需求被“与”在一起：只有被请求的类的 PV 具有和被请求的标签才可以被绑定到 PVC。

> 目前具有非空 selector 的 PVC 不能为其动态配置 PV。

## 声明作为卷

通过声明卷来访问存储。声明必须与使用声明的 pod 存在于相同的命名空间中。集群在 pod 的命名空间中查找声明，并使用它来获取支持声明的 PersistentVolume，该卷将被挂载到主机的 pod 上。

### 命名空间注意点

PersistentVolumes 绑定是唯一的，并且由于 PersistentVolumeClaims 是命名空间对象，因此只能在一个命名空间内挂载具有“多个”模式（ROX、RWX）的声明。

## 原始块卷支持

**`原始块卷的静态配置`**在 v1.9 中作为 alpha 功能引入。由于这个改变，需要一些新的 API 字段来使用该功能。目前，Fibre Channl 是支持该功能的唯一插件。

### 原始块卷作为持久化卷

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### 持久化卷声明请求原始块卷

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

### Pod 规格配置中为容器添加原始块设备

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

> 注意：在为 Pod 增加原始块设备时，在容器中指定设备路径而不是挂载路径。

### 绑定块卷

如果用户通过使用 PersistentVolumeClaim 规范中的 volumeMode 字段来请求原始块卷，则绑定规则与之前不识别该模式规范的版本略有不同。
静态设置的卷的卷绑定矩阵：

| PV volumeMode | PVC volumeMode | Result |
|:------------- | :------------- | ------:|
| unspecified | unspecified | BIND |
| unspecified | Block       | NO BIND|
| unspecified | Filesystem  | BIND |
| Block  | unspecified | NO BIND |
| Block  | Block  | BIND |
| Block  | Filesystem | NO BIND |
| Filesystem | Filesystem | BIND |
| Filesystem | Block | NO BIND |
| Filesystem | unspecified | BIND |

## 编写可移植配置

持久卷配置模板规范：

+ 不要在配置组合中包含 PersistentVolumeClaim 对象（与 Deployment、ConfigMap等一起）。
+ 不要在配置中包含 PersistentVolume 对象，用户实例化配置可能没有创建 PersistentVolume 的权限。
+ 为用户在实例化模板时提供存储类名称的选项。
 + 如果用户提供存储类名称，则将该值放入 persistentVolumeClaim.storageClassName 字段中。如果集群具有由管理员启用的 StorageClass，这将导致 PVC 匹配正确的存储类别。
 + 如果用户未提供存储类名称，则将 persistentVolumeClaim.storageClassName 字段保留为 nil。
 + 这将使用集群中默认的 StorageClass 为用户自动配置 PV。许多集群环境都有默认的 StorageClass，或者管理员可以创建自己的默认 StorageClass。
+ 请注意集群中一段时间之后仍未绑定的 PVC，并向用户展示它们，因为这表示集群可能没有动态存储支持（在这种情况下用户应创建匹配的 PV），或集群没有存储系统（在这种情况下用户不能部署需要 PVC 的配置）。
