---
title: SonarQube on Kubernetes
date: 2018-03-23 19:18:41
categories: DevOps
---
> 本文中PostgreSQL使用 [Ceph RBD](https://acquaai.github.io/2019/09/16/using-ceph-rbd-persistent-storage/) 作为持久卷，SonarQube使用 NFS 作为持久卷。

## 安装 PostgreSQL

### 创建 PostgreSQL Secret

```bash
$ echo -n sonar | base64
c29uYXI=

$ echo -n Passw0rd | base64
UGFzc3cwcmQ=

$ echo -n sonardb | base64
c29uYXJkYg==

#与 echo -n 作用相同
$ tr --delete '\n' < mysql-root-pwd.txt > .strippedmysql-root-pwd.txt && mv .strippedmysql-root-pwd.txt mysql-root-pwd.txt

OR
$ kubectl create secret generic mysql-secrets --from-literal=root-user=root --from-literal=root-password=Passw0rd --from-literal=mysql-database=sonardb --from-literal=mysql-user=sonar --from-literal=mysql-password=sonar -n sonar

OR
$ kubectl create secret generic mysql-root-pwd --from-file=mysql-root-pwd.txt -n sonar
```

<!-- more -->

```bash
$ cat pg-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: sonar
  labels:
    app: postgres
data:
  postgres-db: c29uYXJkYg==
  postgres-user: c29uYXI=
  postgres-password: c29uYXI=
```

```
$ kubectl create -f pg-secrets.yaml
```

```bash
$ kubectl get secrets -n sonar
NAME                  TYPE                                  DATA      AGE
ceph-secret           kubernetes.io/rbd                     1         8d
default-token-fzbwt   kubernetes.io/service-account-token   3         16d
postgres-secrets      Opaque                                3         3d

$ kubectl describe secrets/postgres-secrets  -n sonar
Name:         postgres-secrets
Namespace:    sonar
Labels:       app=postgres
Annotations:  <none>

Type:  Opaque

Data
====
postgres-db:        7 bytes
postgres-password:  5 bytes
postgres-user:      5 bytes
```

### 部署 PostgreSQL

```json
$ cat pg-statefulset.yaml 
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: sonar
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: postgres
  namespace: sonar
  labels:
    app: postgres
spec:
  serviceName: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: postgresql
        image: 10.0.77.16/library/postgres:10.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
          name: postgresql
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres-user
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres-db
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-volume
          mountPath: /var/lib/postgresql/data
      imagePullSecrets: 
        - name: registrykey
  volumeClaimTemplates:
  - metadata:
      name: postgres-volume
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ceph-postgres
      resources:
        requests:
          storage: 10Gi
```

```bash
$ kubectl create -f pg-statefulset.yaml
```

> volumeClaimTemplates中定义了PVC，并指定了`storageClassName: "ceph-postgres"`，会根据PVC动态创建Pod所需的PV。

```bash
$ kubectl get pvc -n sonar
NAME                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
postgres-volume-postgres-0   Bound     pvc-df69da97-2e7a-11e8-b7b9-0050568879ea   10Gi       RWO            ceph-postgres   5m

$ kubectl get pv -n sonar
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                              STORAGECLASS    REASON    AGE
pvc-df69da97-2e7a-11e8-b7b9-0050568879ea   10Gi       RWO            Delete           Bound     sonar/postgres-volume-postgres-0   ceph-postgres             7m

$ kubectl describe persistentvolume  pvc-df69da97-2e7a-11e8-b7b9-0050568879ea
Name:            pvc-df69da97-2e7a-11e8-b7b9-0050568879ea
Labels:          <none>
Annotations:     kubernetes.io/createdby=rbd-dynamic-provisioner
                 pv.kubernetes.io/bound-by-controller=yes
                 pv.kubernetes.io/provisioned-by=kubernetes.io/rbd
StorageClass:    ceph-postgres
Status:          Bound
Claim:           sonar/postgres-volume-postgres-0
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        10Gi
Message:         
Source:
    Type:          RBD (a Rados Block Device mount on the host that shares a pod's lifetime)
    CephMonitors:  [10.0.77.17:6789]
    RBDImage:      kubernetes-dynamic-pvc-df734e53-2e7a-11e8-93af-00505688047e
    FSType:        xfs
    RBDPool:       rbd
    RadosUser:     admin
    Keyring:       /etc/ceph/keyring
    SecretRef:     &{ceph-secret }
    ReadOnly:      false
Events:            <none>
```

##  部署 Sonarqube

```json
$ cat sonar-rc.yaml
apiVersion: v1
kind: Service
metadata:
  name: sonar
  namespace: sonar
spec:
  type: NodePort
  ports:
  - port: 9000
    protocol: TCP
    nodePort: 30001
  selector:
    name: sonar
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: sonar
  namespace: sonar
  labels:
    name: sonar
spec:
  replicas: 1
  selector:
    name: sonar
  template:
    metadata:
      labels:
        name: sonar
    spec:
      containers:
      - name: sonar
        image: 10.0.77.16/library/sonarqube:6.7
        env:
        - name: SONARQUBE_JDBC_USERNAME
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres-user
        - name: SONARQUBE_JDBC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: postgres-password
        - name: SONARQUBE_JDBC_URL
          value: jdbc:postgresql://postgres:5432/sonardb
        ports:
        - containerPort: 9000
        volumeMounts:
        - mountPath: "/opt/sonarqube/conf"
          name: sonar-conf
        - mountPath: "/opt/sonarqube/data"
          name: sonar-data
        - mountPath: "/opt/sonarqube/extensions"
          name: sonar-extensions
        - mountPath: "/opt/sonarqube/logs"
          name: sonar-logs
      volumes:
        - name: sonar-conf
          nfs:
            server: 10.0.77.16
            path: "/nfs/sonar/conf"
        - name: sonar-data
          nfs:
            server: 10.0.77.16
            path: "/nfs/sonar/data"
        - name: sonar-extensions
          nfs:
            server: 10.0.77.16
            path: "/nfs/sonar/extensions"
        - name: sonar-logs
          nfs:
            server: 10.0.77.16
            path: "/nfs/sonar/logs"    
```

```bash
$ kubectl create -f sonar-rc.yaml
```

使用默认用户、密码 admin/admin 访问SonarQube http://k8s-node-ip:30001

## SonarQube Plugins
 
[Plugin Library](https://docs.sonarqube.org/display/PLUG)

SonarQube v6.7预装好的Plugins如下：

```bash
$ kubectl get pods -n sonar
NAME          READY     STATUS    RESTARTS   AGE
postgres-0    1/1       Running   0          15h
sonar-wmvnf   1/1       Running   0          15h

$ kubectl exec -n sonar sonar-wmvnf -it -- bash
root@sonar-wmvnf:/opt/sonarqube/extensions/plugins# ls -l
total 40464
-rw-r--r-- 1 sonarqube sonarqube 2703958 Nov  6 08:12 sonar-csharp-plugin-6.5.0.3766.jar
-rw-r--r-- 1 sonarqube sonarqube 1618672 Nov  6 08:12 sonar-flex-plugin-2.3.jar
-rw-r--r-- 1 sonarqube sonarqube 6759535 Nov  6 15:31 sonar-java-plugin-4.15.0.12310.jar
-rw-r--r-- 1 sonarqube sonarqube 3355702 Nov  6 08:12 sonar-javascript-plugin-3.2.0.5506.jar
-rw-r--r-- 1 sonarqube sonarqube 3022870 Nov  6 16:50 sonar-php-plugin-2.11.0.2485.jar
-rw-r--r-- 1 sonarqube sonarqube 4024311 Nov  6 08:12 sonar-python-plugin-1.8.0.1496.jar
-rw-r--r-- 1 sonarqube sonarqube 3625962 Nov  6 08:12 sonar-scm-git-plugin-1.3.0.869.jar
-rw-r--r-- 1 sonarqube sonarqube 6680471 Nov  6 08:12 sonar-scm-svn-plugin-1.6.0.860.jar
-rw-r--r-- 1 sonarqube sonarqube 2250667 Nov  6 08:12 sonar-typescript-plugin-1.1.0.1079.jar
-rw-r--r-- 1 sonarqube sonarqube 7368250 Nov  6 08:12 sonar-xml-plugin-1.4.3.1027.jar
```

## SonarQube Scanner

[Analyzing with SonarQube Scanner](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner)

```bash
root@sonar-wmvnf:~# cd /opt/sonarqube/extensions/downloads
root@sonar-wmvnf:/opt/sonarqube/extensions/downloads# wget https://sonarsource.bintray.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-3.1.0.1141-linux.zip
root@sonar-wmvnf:/opt/sonarqube/extensions/downloads# unzip sonar-scanner-cli-3.1.0.1141-linux.zip -d /opt/sonarqube
root@sonar-wmvnf:/opt/sonarqube# mv sonar-scanner-3.1.0.1141-linux sonar-scanner
root@sonar-wmvnf:/opt/sonarqube# chown -R sonarqube:sonarqube sonar-scanner/
```

增加 SonarQube Scanner 的环境变量

```bash
$ kubectl cp sonar/sonar-wmvnf:/etc/profile ./
tar: Removing leading `/' from member names

$ vi profile
......
/opt/sonarqube/sonar-scanner/bin
......
```

```bash
$ kubectl cp ./profile sonar/sonar-wmvnf:/etc/

root@sonar-wmvnf:/opt/sonarqube# source /etc/profile
root@sonar-wmvnf:/opt/sonarqube# sonar-scanner -h
INFO: 
INFO: usage: sonar-scanner [options]
INFO: 
INFO: Options:
INFO:  -D,--define <arg>     Define property
INFO:  -h,--help             Display help information
INFO:  -v,--version          Display version information
INFO:  -X,--debug            Produce execution debug output
```

配置 SonarQube Scanner 属性

```bash
$ cat sonar-scanner.properties

sonar.host.url=http://localhost:9000
sonar.sourceEncoding=UTF-8
```

```bash
$ kubectl cp ./sonar-scanner.properties sonar/sonar-wmvnf:/opt/sonarqube/sonar-scanner/conf/sonar-scanner.properties
```

配置 Python 代码分析 

```bash
$ cat sonar-project.properties
sonar.projectKey=my:python
sonar.projectName=python
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8

sonar.language=py
#sonar.python.pylint=/usr/bin/pylint
#sonar.python.pylint_config=.pylintrc
#sonar.python.pylint.reportPath=./pylint-report.txt

```

```bash
root@sonar-wmvnf:/opt/sonarqube/conf# mkdir python

$ kubectl cp ./sonar-project.properties sonar/sonar-wmvnf:/opt/sonarqube/conf/python/sonar-project.properties

root@sonar-wmvnf:/opt/sonarqube/conf# chown -R sonarqube:sonarqube python
```

执行分析

```bash
root@sonar-wmvnf:/opt/sonarqube/conf/python# sonar-scanner -Dsonar.projectKey=my:python -Dsonar.sources=.

INFO: Scanner configuration file: /opt/sonarqube/sonar-scanner/conf/sonar-scanner.properties
INFO: Project root configuration file: /opt/sonarqube/conf/python/sonar-project.properties
INFO: SonarQube Scanner 3.1.0.1141
......
......
INFO: ANALYSIS SUCCESSFUL, you can browse http://localhost:9000/dashboard/index/my:python
INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
INFO: More about the report processing at http://localhost:9000/api/ce/task?id=AWJW_N6U1qxTjZHyQmBu
INFO: Task total time: 4.424 s
INFO: ------------------------------------------------------------------------
INFO: EXECUTION SUCCESS
INFO: ------------------------------------------------------------------------
INFO: Total time: 6.144s
INFO: Final Memory: 11M/335M
INFO: ------------------------------------------------------------------------
```

![](/images/sonarPython.png)

## Jenkins集成Sonarqube

+ [Analyzing with SonarQube Scanner for Jenkins](Analyzing with SonarQube Scanner for Jenkins)

**Reference**

+ [使用Ceph做持久化存储创建MySQL集群](https://jimmysong.io/kubernetes-handbook/practice/using-ceph-for-persistent-storage.html)
+ [Running Highly Available WordPress with MySQL on Kubernetes](http://rancher.com/running-highly-available-wordpress-mysql-kubernetes/)
+ [NFS PV(C)](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/nfs)
