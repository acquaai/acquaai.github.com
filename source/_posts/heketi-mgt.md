---
title: Heketi Management
date: 2019-09-29 09:38:55
categories: DevOps
---
Heketi provides a RESTful management interface which can be used to manage the life cycle of GlusterFS volumes. With Heketi, cloud services like OpenStack Manila, Kubernetes, and Openshift can dynamically provision GlusterFS volumes.

If Heketi is not setup with authentication, then use curl to verify the configuration:

```zsh
curl http://<server:port>/hello
Hello from Heketi
```

You can also verify the configuration using the heketi-cli when authentication is enabled:

```zsh
heketi-cli -server http://<server:port> -user <user> -secret <key> cluster list
```

**Get heketi secret name**

```zsh
$ kubectl get secret
NAME                                 TYPE                                  DATA   AGE
heketi-config-secret                 Opaque                                3      17h
```

**Get heketi secret for Admin access in order to use API as Admin**

<!-- more -->

```json
$ kubectl get secret heketi-config-secret -o jsonpath --template '{.data.heketi\.json}' | base64 -d
{
        "_port_comment": "Heketi Server Port Number",
        "port" : "8080",

        "_use_auth": "Enable JWT authorization. Please enable for deployment",
        "use_auth" : false,

        "_jwt" : "Private keys for access",
        "jwt" : {
                "_admin" : "Admin has access to all APIs",
                "admin" : {
                        "key" : ""
                },
                "_user" : "User only has access to /volumes endpoint",
                "user" : {
                        "key" : ""
                }
        },

        "_glusterfs_comment": "GlusterFS Configuration",
        "glusterfs" : {

                "_executor_comment": "Execute plugin. Possible choices: mock, kubernetes, ssh",
                "executor" : "kubernetes",

                "_db_comment": "Database file name",
                "db" : "/var/lib/heketi/heketi.db",

                "kubeexec" : {
                        "rebalance_on_expansion": true
                },

                "sshexec" : {
                        "rebalance_on_expansion": true,
                        "keyfile" : "/etc/heketi/private_key",
                        "port" : "22",
                        "user" : "root",
                        "sudo" : false
                }
        },

        "backup_db_to_kube_secret": false
}
```

Verifying GlusterFS resources

```zsh
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
glusterfs-gmx44           1/1     Running   0          18h
glusterfs-gzpvg           1/1     Running   0          18h
glusterfs-rb7t7           1/1     Running   0          18h
heketi-8658674597-bztc8   1/1     Running   0          18h
```

```zsh
$ kubectl exec -it heketi-8658674597-bztc8 -- heketi-cli \ 
--insecure-tls --user admin --secret QVFEQUNWWmRmTUxBSkJBQXlSV88rUm11RzJSb0J2Tk9SVllSaGc9PQ== \ 
cluster list

Clusters:
Id:30058720288f89c7cf3c2016bdc905b8 [file][block]

$ kubectl exec -it heketi-8658674597-bztc8 -- heketi-cli \ 
--user admin --secret QVFEQUNWWmRmTUxBSkJBQXlSV88rUm11RzJSb0J2Tk9SVllSaGc9PQ== \ 
cluster info 30058720288f89c7cf3c2016bdc905b8

Cluster id: 30058720288f89c7cf3c2016bdc905b8
Nodes:
3aa460b30b825f44fab97737ca01ee1f
46e9bc2a495f82b9411f2a0eb6e701d9
e5930a2a72e43267c104592fb358231b
Volumes:
7c7d1aa2d2a6afaaf4c3f008b2ff6c9f
924f29533b8ba0959e2c2b43ef2cc44a
a2514cec343f8e324488e057c9fd7358
f4bcd5ba2474caae96c45a2a5c77d28c
Block: true

File: true
```

```zsh
$ kubectl exec -it heketi-8658674597-bztc8 -- heketi-cli \ 
--user admin --secret QVFEQUNWWmRmTUxBSkJBQXlSV88rUm11RzJSb0J2Tk9SVllSaGc9PQ== \ 
node info 3aa460b30b825f44fab97737ca01ee1f

Node Id: 3aa460b30b825f44fab97737ca01ee1f
State: online
Cluster Id: 30058720288f89c7cf3c2016bdc905b8
Zone: 1
Management Hostname: w-192-168-16-16
Storage Hostname: 192.168.16.16
Devices:
Id:37b799f5d213aacac0a46f94d53b359b   Name:/dev/sdb            State:online    Size (GiB):3351    Used (GiB):906     Free (GiB):2445    Bricks:4      

$ kubectl exec -it heketi-8658674597-bztc8 -- heketi-cli \ 
--user admin --secret QVFEQUNWWmRmTUxBSkJBQXlSV88rUm11RzJSb0J2Tk9SVllSaGc9PQ== \ 
device info 37b799f5d213aacac0a46f94d53b359b

Device Id: 37b799f5d213aacac0a46f94d53b359b
State: online
Size (GiB): 3351
Used (GiB): 906
Free (GiB): 2445
Create Path: /dev/sdb
Physical Volume UUID: UX4c4K-YWfv-u0kh-9CQA-wTQz-yNvb-vy8TX8
Known Paths: /dev/sdb
Bricks:
Id:0d98000f85540d0854b0db455706c2ae   Size (GiB):300     Path: /var/lib/heketi/mounts/vg_37b799f5d213aacac0a46f94d53b359b/brick_0d98000f85540d0854b0db455706c2ae/brick
Id:2fad4ddf0a59da0763c5f083c15f4ad6   Size (GiB):300     Path: /var/lib/heketi/mounts/vg_37b799f5d213aacac0a46f94d53b359b/brick_2fad4ddf0a59da0763c5f083c15f4ad6/brick
Id:7d63092c9cb72c28af6fbcc8e920f52d   Size (GiB):300     Path: /var/lib/heketi/mounts/vg_37b799f5d213aacac0a46f94d53b359b/brick_7d63092c9cb72c28af6fbcc8e920f52d/brick
Id:ec52206e5ca0aeec17dea6caa33eb9ab   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_37b799f5d213aacac0a46f94d53b359b/brick_ec52206e5ca0aeec17dea6caa33eb9ab/brick
```

```zsh
$  kubectl exec -it glusterfs-gmx44 -- gluster volume list
heketidbstorage
vol_7c7d1aa2d2a6afaaf4c3f008b2ff6c9f
vol_a2514cec343f8e324488e057c9fd7358
vol_f4bcd5ba2474caae96c45a2a5c77d28c

$ kubectl exec -it glusterfs-gmx44 -- gluster volume \ 
info vol_a2514cec343f8e324488e057c9fd7358
 
Volume Name: vol_a2514cec343f8e324488e057c9fd7358
Type: Replicate
Volume ID: 6bc5b5c5-a391-496a-9e9f-f064642d73d8
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 192.168.16.14:/var/lib/heketi/mounts/vg_16d9f54959b1fd03aab539861402098b/brick_3566e7f014deeba68637e7a37b28ea5e/brick
Brick2: 192.168.16.15:/var/lib/heketi/mounts/vg_aa2919673d46d0f059d4f99bc52c2fd1/brick_d3cf2bca3b5c45455af3b973ec1ddafd/brick
Brick3: 192.168.16.16:/var/lib/heketi/mounts/vg_37b799f5d213aacac0a46f94d53b359b/brick_0d98000f85540d0854b0db455706c2ae/brick
Options Reconfigured:
user.heketi.id: a2514cec343f8e324488e057c9fd7358
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

[Heketi Documentation](https://github.com/heketi/heketi/tree/master/docs/)
