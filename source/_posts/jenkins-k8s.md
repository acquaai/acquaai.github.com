---
title: Jenkins on Kubernetes
date: 2018-03-28 17:00:11
categories: DevOps
---
![](/images/jenkinsonk8s1.png)

<!-- more -->

> 由于部分 Jenkins Plugins 安装需要越墙，加之 Plugins 之间的依赖关系若全手动安装会把人逼疯掉，此时使用官方 jenkinsci/blueocean 镜像成为一个明智选择，此镜像已包含官方推荐的插件。Jenkins Slave 使用 jenkins/jnlp-slave:latest 镜像。

## 部署 Jenkins

```json
$ cat jenkins-pvc.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: jenkins
type: "kubernetes.io/rbd"  
data:
  key: QVFDd2hLZGFJYktSSHhBQVlmQ21vaitWUnNmUVhTczA3ODRLb3c9PQ==
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
   name: ceph-jenkins
   namespace: jenkins
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.0.77.17:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: jenkins
  pool: k8s
  userId: admin
  userSecretName: ceph-secret
  fsType: xfs
  imageFormat: "2"
  imageFeatures: "layering"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-master-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-jenkins  
  resources:
    requests:
      storage: 50Gi
```

```json
$ cat jenkins-svc.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: jenkins
  name: jenkins-admin
  namespace: jenkins
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins-default
rules:
- apiGroups: ["","extensions","app"]
  resources: ["pods","pods/exec","deployments","replicasets"]
  verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
  labels:
    k8s-app: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-default
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: jenkins
---
#Generally, don't need to create a service, the service here only for jnlp connect.
kind: Service
apiVersion: v1
metadata:
  labels:
    app: jenkins-master
  name: jenkins-service
  namespace: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      name: jenkins
    - port: 50000
      name: agent
  selector:
    app: jenkins-master
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins
  namespace: jenkins
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: jenkins-service
          servicePort: 80
    host: jenkins.k8s.com #Define your's.
```

```json
$ cat jenkins-deployment.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins-master
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins-master
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
      serviceAccountName: jenkins-admin
      containers:
      - name: jenkins-master
        image: 10.0.77.16/library/jenkinsci/blueocean
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: jenkins
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 0.5
            memory: 500Mi
        volumeMounts:
          - name: docker
            mountPath: /var/run/docker.sock
          - name: jenkins-persistent-storage
            mountPath: /var/jenkins_home
        env:
          - name: JAVA_OPTS
            value: "-Duser.timezone=Asia/Shanghai"
      volumes:
      - name: docker
        hostPath:
          path: /var/run/docker.sock
      - name: jenkins-persistent-storage
        persistentVolumeClaim:
          claimName: jenkins-master-pvc
```

```bash
$ kubectl get svc -n jenkins
NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
jenkins-service   NodePort   10.254.112.72   <none>        8080:30514/TCP,50000:32589/TCP   4h

$ kubectl get pods -n jenkins
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-5d659f6b99-nwpxk   1/1       Running   0          3m

$ kubectl exec -n jenkins jenkins-5d659f6b99-nwpxk -it -- bash
bash-4.4$ cat /var/jenkins_home/secrets/initialAdminPassword
5ad866a38c674a66a2a2fd6adc9702cd
```

修改本地hosts文件解析`jenkins.k8s.com:30514`到k8s集群节点IP，配置Jenkins。

## 配置 Kubernetes 插件

手动下载**kubernetes、kubernetes-credentials**插件并安装。

![](/images/jenkinsonk8s2.png)

## 创建Pipeline Job

Bypass for now.

**References**

+ [Kuberbetes-- 利用Jenkins在Kubernetes中实践CI/CD](https://zhangchenchen.github.io/2017/12/17/achieve-cicd-in-kubernetes-with-jenkins/)
+ [jenkins with pipeline on kubernetes](https://kevinguo.me/2017/12/27/jenkins-on-kubernetes-with-pipeline/)
