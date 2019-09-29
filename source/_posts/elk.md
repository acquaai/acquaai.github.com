---
title: ELK Stack 7 on Kubernetes
date: 2019-09-29 11:37:38
categories: DevOps
---
## What is the ELK Stack?

"ELK" is the acronym for three open source projects: Elasticsearch, Logstash, and Kibana. Elasticsearch is a search and analytics engine. Logstash is a server‑side data processing pipeline that ingests data from multiple sources simultaneously, transforms it, and then sends it to a "stash" like Elasticsearch. Kibana lets users visualize data with charts and graphs in Elasticsearch.


## Tips

This post reference to the [doc](https://flywzj.com/blog/es/). But **Pod memory resource limit** like:

```yaml
......
  - name: ES_JAVA_OPTS
    value: "-Xms2g -Xmx2g"
resources:
  limits:
    cpu: '1'
    memory: 2Gi
  requests:
    cpu: '1'
    memory: 2Gi
......
```

You have to: resources:`limits:memory: 2Gi` > `value: "-Xms2g -Xmx2g"`, or `OOM`.


## Deploying

<!-- more -->

The complete file is [here](https://github.com/acquaai/ELK-Stack/tree/master/yaml). Run in sequence, 

```zsh
kubectl apply -f ns-kafka.yaml

kubectl apply -f es-master-config.yaml
kubectl apply -f es-master-service.yaml
kubectl apply -f es-master.yaml

kubectl apply -f es-data-config.yaml
kubectl apply -f es-data-service.yaml
kubectl apply -f es-data.yaml

kubectl apply -f es-ingest-config.yaml
kubectl apply -f es-ingest-service.yaml
kubectl apply -f es-ingest.yaml

kubectl apply -f logstash-pipeline-config.yaml
kubectl apply -f logstash.yaml
kubectl apply -f logstash-service.yaml

kubectl apply -f kibana-config.yaml
kubectl apply -f kibana-service.yaml
kubectl apply -f kibana.yaml
kubectl apply -f kibana-ingress.yaml

kubectl apply -f curator-config.yaml
kubectl apply -f curator.yaml
```


## Verifying

+ elk pod

```zsh
$ kubectl get pod -n kafka
NAME          READY   STATUS    RESTARTS   AGE
es-data-0     1/1     Running   0          4m6s
es-data-1     1/1     Running   0          3m47s
es-data-2     1/1     Running   0          3m28s
es-ingest-0   1/1     Running   0          52s
es-ingest-1   1/1     Running   0          47s
es-ingest-2   1/1     Running   0          42s
es-master-0   1/1     Running   0          6m29s
es-master-1   1/1     Running   0          6m25s
es-master-2   1/1     Running   0          6m22s
```

+ service

```zsh
$ kubectl get svc -n kafka | grep es
es-data          ClusterIP      None             <none>        9300/TCP         3d5h
es-ingest        LoadBalancer   10.98.208.109    <pending>     9200:30982/TCP   3d5h
es-master        ClusterIP      None             <none>        9300/TCP         3d5h
```

+ elk information

```zsh
$ curl http://172.31.16.11:30982
{
  "name" : "es-ingest-2",
  "cluster_name" : "es-kafka",
  "cluster_uuid" : "8R8FUTggQWmBwI2PKMFl2w",
  "version" : {
    "number" : "7.3.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "1c1faf1",
    "build_date" : "2019-09-06T14:40:30.409026Z",
    "build_snapshot" : false,
    "lucene_version" : "8.1.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

+ elk nodes

```zsh
$ curl http://172.31.16.11:30982/_cat/nodes?v
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.244.83.116            1           8   2    0.22    0.39     0.32 i         -      es-ingest-1
10.244.244.20            8           9   4    0.47    0.75     0.69 m         *      es-master-0
10.244.83.115            3           8   3    0.22    0.39     0.32 d         -      es-data-1
10.244.244.21            1           9   3    0.47    0.75     0.69 i         -      es-ingest-0
10.244.84.27             6           9   4    0.15    0.47     0.67 m         -      es-master-1
10.244.83.114            6           8   3    0.22    0.39     0.32 m         -      es-master-2
10.244.84.26             4           9   3    0.15    0.47     0.67 d         -      es-data-2
10.244.244.19            2           9   3    0.47    0.75     0.69 d         -      es-data-0
10.244.84.31             1           9   2    0.15    0.47     0.67 i         -      es-ingest-2
```

+ elk indices

```zsh
$ curl http://172.31.16.11:30982/_cat/indices?v
health status index                   uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_task_manager    DGLMxRtcQJWl_FTcSsOwJg   1   1          2            0     59.6kb         29.8kb
green  open   .kibana_1               XzaoAq0CQXmvwfGuesZCUg   1   1         10            3    108.5kb         54.2kb
green  open   nginx-access-2019.09.27 HzoYMc11QI68DDBtAmNU3g   1   1    3304714            0      4.1gb          1.9gb
green  open   nginx-access-2019.09.16 hj229YYbQXWWVzPS5cWp8A   2   1     269961            0    337.2mb        168.1mb
green  open   nginx-access-2019.09.28 VhVuxXQvSR-1GLWjKgTJsA   1   1   37900045            0     31.2gb           17gb
green  open   nginx-access-2019.09.17 zsqJ4RcWQ5u7n59fcSiDww   2   1     534824            0    693.3mb        342.6mb
green  open   nginx-access-2019.09.29 jrJqOpZTQ_C3jd3oD50oRQ   2   1    6986701            0      4.6gb          2.2gb
green  open   nginx-access-2019.09.18 xvdO_S_5S4Kf30FexhjcMA   2   1     835914            0        1gb          531mb
green  open   nginx-access-2019.09.19 LW45Dw6FRZ-PCRIPH1eO8g   2   1     805261            0        1gb        524.9mb
green  open   nginx-access-2019.09.23 KkXMoqHhQ3iv9XOsDu5C5A   2   1     823928            0        1gb        523.4mb
green  open   nginx-access-2019.09.24 9MM-wjLVTrm3UVQlfdCT4w   2   1     815697            0        1gb        527.3mb
green  open   nginx-access-2019.09.25 uZU7v_kZQ3ObKXa3XWN1tg   2   1     808426            0        1gb        522.6mb
green  open   nginx-access-2019.09.15 Ma8h8GmPQ7O8e6rXmfNGVg   2   1     521880            0    625.2mb        318.2mb
green  open   nginx-access-2019.09.20 iRIpy6dJQCagViSf-uxwWw   2   1     843735            0        1gb        525.3mb
green  open   nginx-access-2019.09.21 BvKQ8fsfTRm6nd2MzD7YFQ   2   1     208548            0    277.7mb        138.9mb
```

+ elk health 

```zsh
$ curl http://172.31.16.11:30982/_cat/health?v
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1569726860 03:14:20  es-kafka green           9         3     52  26    0    0        0             0                  -                100.0%
```

```
$ kubectl get pod -n kafka | egrep "es|logstash|kibana"
es-data-0                   1/1     Running   0          4h
es-data-1                   1/1     Running   0          4h
es-data-2                   1/1     Running   0          3h59m
es-ingest-0                 1/1     Running   0          3h57m
es-ingest-1                 1/1     Running   0          3h57m
es-ingest-2                 1/1     Running   0          3h57m
es-master-0                 1/1     Running   0          4h2m
es-master-1                 1/1     Running   0          4h2m
es-master-2                 1/1     Running   0          4h2m
kibana-856dcb8975-948c9     1/1     Running   0          25m
logstash-6856879649-5gzq4   1/1     Running   0          62m
```


## ELK index template for Nginx

```json
PUT _template/nginx
{
......
}
```


## Customized Logstash Image

Add `GeoLite2-City.mmdb` to image.

```zsh
$ cat Dockerfile 
FROM registry.xxx.com/library/logstash:7.3.2
COPY --chown=1000:root GeoLite2-City.mmdb /usr/share/GeoIP2/
```

```zsh
$ docker build . --no-cache -t registry.xxx.com/library/logstash-geoip:7.3.2
Sending build context to Docker daemon  53.74MB
Step 1/2 : FROM registry.xxx.com/library/logstash:7.3.2
 ---> ed2f8442f606
Step 2/2 : COPY --chown=1000:root GeoLite2-City.mmdb /usr/share/GeoIP2/
 ---> 5900fad8a37a
Successfully built 5900fad8a37a
Successfully tagged registry.xxx.com/library/logstash-geoip:7.3.2
```


## Curator

Elasticsearch Curator helps you curator, or manage, your Elasticsearch indices and snapshots.

```zsh
kubectl get cronjob -n kafka
NAME      SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
curator   0 1 * * *   False     0        11h             2d2h

$ kubectl get jobs -n kafka
NAME                 COMPLETIONS   DURATION   AGE
curator-1569643200   1/1           3m11s      23h
curator-1569651300   1/1           3m20s      21h
curator-1569684600   1/1           64s        11h
```


**Ref**

+ [Kubernetes ELK: How to Run HA Elasticsearch (ELK) on Google Kubernetes Engine](https://portworx.com/run-ha-elasticsearch-elk-google-kubernetes-engine/)
+ [ELK stack in k8s cluster](https://medium.com/@tharangarajapaksha/elk-stack-in-k8s-cluster-13bb509185e0)
+ [Deployment of full-scale ELK stack to Kubernetes](https://hackernoon.com/deployment-of-full-scale-elk-stack-to-kubernetes-6f38f6c57c55)
