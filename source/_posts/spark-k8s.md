---
title: Spark on Kubernetes
date: 2018-03-29 15:35:28
categories: DevOps
---
> Apache Spark™ is a fast and general engine for large-scale data processing.

```
gcr.io/google_containers/spark:1.5.2_v1
--->
10.0.77.16/library/spark:1.5.2_v1

elsonrodriguez/spark-ui-proxy:1.0
--->
10.0.77.16/library/spark-ui-proxy:1.0

gcr.io/google_containers/zeppelin:v0.5.6_v1
--->
10.0.77.16/library/zeppelin:v0.5.6_v1
```

[YAML Files](https://github.com/acquaai/K8S/tree/master/yaml/Spark-K8s)

<!-- more -->

## Create Namespace

```bash
$ kubectl create -f spark-namespace.yaml
```

## Deploy Master

```bash
$ kubectl create -f spark-master-rc.yaml
$ kubectl create -f spark-master-svc.yaml
```

```bash
$ kubectl get pod -n spark-cluster
NAME                            READY     STATUS    RESTARTS   AGE
spark-master-controller-h9rvd   1/1       Running   0          1m

$ kubectl logs spark-master-controller-h9rvd -n spark-cluster
18/03/29 03:32:09 INFO Master: Registered signal handlers for [TERM, HUP, INT]
18/03/29 03:32:10 INFO SecurityManager: Changing view acls to: root
18/03/29 03:32:10 INFO SecurityManager: Changing modify acls to: root
18/03/29 03:32:10 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(root); users with modify permissions: Set(root)
18/03/29 03:32:11 INFO Slf4jLogger: Slf4jLogger started
18/03/29 03:32:11 INFO Remoting: Starting remoting
18/03/29 03:32:11 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkMaster@spark-master:7077]
18/03/29 03:32:11 INFO Utils: Successfully started service 'sparkMaster' on port 7077.
18/03/29 03:32:11 INFO Master: Starting Spark master at spark://spark-master:7077
18/03/29 03:32:11 INFO Master: Running Spark version 1.5.2
18/03/29 03:32:22 INFO Utils: Successfully started service 'MasterUI' on port 8080.
18/03/29 03:32:22 INFO MasterWebUI: Started MasterWebUI at http://172.30.48.2:8080
18/03/29 03:32:22 INFO Utils: Successfully started service on port 6066.
18/03/29 03:32:22 INFO StandaloneRestServer: Started REST server for submitting applications on port 6066
18/03/29 03:32:22 INFO Master: I have been elected leader! New state: ALIVE
```

[Spark UI Proxy](https://github.com/aseigneurin/spark-ui-proxy)

```bash
$ kubectl create -f spark-ui-proxy-rc.yaml
$ kubectl create -f spark-ui-proxy-svc.yaml

$ kubectl get svc -n spark-cluster
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
spark-master     ClusterIP   10.254.137.145   <none>        7077/TCP,8080/TCP   1h
spark-ui-proxy   NodePort    10.254.48.115    <none>        80:30080/TCP        2m
```

![](/images/spark1-k8s.png)

## Deploy Workers

```bash
$ kubectl create -f spark-worker-rc.yaml
```

## [Zeppelin](http://zeppelin.apache.org)

```bash
$ kubectl create -f zeppelin-rc.yaml
$ kubectl create -f zeppelin-svc.yaml

$ kubectl exec zeppelin-controller-pnn5h -it -n spark-cluster pyspark

Python 2.7.9 (default, Mar  1 2015, 12:57:24) 
[GCC 4.9.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 1.5.2
      /_/

Using Python version 2.7.9 (default, Mar  1 2015 12:57:24)
SparkContext available as sc, HiveContext available as sqlContext.
>>> 
```

![](/images/spark2-k8s.png)

**References**

+ [Spark](https://feisky.gitbooks.io/kubernetes/zh/machine-learning/spark.html)
