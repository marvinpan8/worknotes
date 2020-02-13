https://blog.51cto.com/newfly/2139393

kubernetes中部署Heketi和GlusterFS(二)
在上一节中，Heketi的部署方式还不能用于生产环境，因为Heketi Pod的数据并没有持久化，容易导致heketi的数据丢失，Heketi的数据保存在**/var/lib/heketi/heketi.db**文件中，因此需要把此目录挂载到GlusterFS分布式存储中。

按照上一节的步骤，执行heketi-cli topology load --json=topology-sample.json

```bash
$ echo $HEKETI_CLI_SERVER
http://10.254.49.43:8080

$ heketi-cli topology load --json=topology-sample.json
Creating cluster ... ID: 5b930ef6081fd22e895c25a3dfb0c516
    Allowing file volumes on cluster.
    Allowing block volumes on cluster.
    Creating node 10.30.1.15 ... ID: b120572be40db6c1d979c3903876430b
        Adding device /dev/sdb ... OK
    Creating node 10.30.1.16 ... ID: 7ce13ffc5eabe64a3791e93233fd3c1a
        Adding device /dev/sdb ... OK
    Creating node 10.30.1.17 ... ID: f9abdc2e5d4cfa17c035a97f984a1a3b
        Adding device /dev/sdb ... OK
```

紧接着执行：将生成heketi-storage.json文件。

```bash
$ heketi-cli setup-openshift-heketi-storage
Saving heketi-storage.json
```

> 如果在运行setup-openshift-heketi-storage子命令时heketi-cli报告“无空间”错误：
> $ heketi-cli setup-openshift-heketi-storage
> Error: Failed to allocate new volume: No space
> 则可能无意中运行topology load命令的时候，服务端和heketi-cli的版本不匹配造成的。
>
> 1. 停止正在运行的heketi pod：
>    kubectl scale deployment deploy-heketi --replicas=0
> 2. 手动删除存储块设备中的任何签名：
>    加载拓扑的操作是在gluster 中添加了Peer，所以需要手动detach peer
> 3. 然后继续运行heketi pod：
>    kubectl scale deployment deploy-heketi --replicas=1。
> 4. 用匹配版本的heketi-cli重新加载拓扑，然后重试该步骤。

执行完后，查看Pod deploy-heketi日志信息，看看做了哪些事：

```bash
#只截取了部分日志，基本操作就是进入到各个glusterfs Pod创建brick目录及创建一个副本为3的Replicate volume, volume名为heketidbstorage
[kubeexec] DEBUG 2018/07/09 07:07:23 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-8qrpt Command: mkdir -p /var/lib/heketi/mounts/vg_a146220fd3f761e8da2be784523ce07e/brick_6f0ce82692e70ce5ae2ec55a60f237c6
Result: 
[kubeexec] DEBUG 2018/07/09 07:07:23 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-c4859 Command: mkdir -p /var/lib/heketi/mounts/vg_19584b16bc8f21b87662b27b551652fb/brick_abcb32853351840ee82a95693cbb63b4
Result: 
[kubeexec] DEBUG 2018/07/09 07:07:23 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-25cm8 Command: mkdir -p /var/lib/heketi/mounts/vg_9534f15dd9f0822ad454140d13c660a5/brick_ba4091b858d94a088b21a582d8d4abaa

[kubeexec] DEBUG 2018/07/09 07:07:26 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-8qrpt Command: mkdir /var/lib/heketi/mounts/vg_a146220fd3f761e8da2be784523ce07e/brick_6f0ce82692e70ce5ae2ec55a60f237c6/brick

[kubeexec] DEBUG 2018/07/09 07:07:26 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-8qrpt Command: mkdir /var/lib/heketi/mounts/vg_a146220fd3f761e8da2be784523ce07e/brick_6f0ce82692e70ce5ae2ec55a60f237c6/brick

[kubeexec] DEBUG 2018/07/09 07:07:26 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-c4859 Command: mkdir /var/lib/heketi/mounts/vg_19584b16bc8f21b87662b27b551652fb/brick_abcb32853351840ee82a95693cbb63b4/brick

[kubeexec] DEBUG 2018/07/09 07:07:26 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-25cm8 Command: mkdir /var/lib/heketi/mounts/vg_9534f15dd9f0822ad454140d13c660a5/brick_ba4091b858d94a088b21a582d8d4abaa/brick
Result: 
[cmdexec] INFO 2018/07/09 07:07:26 Creating volume heketidbstorage replica 3
[kubeexec] DEBUG 2018/07/09 07:07:27 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-25cm8 Command: gluster --mode=script volume create heketidbstorage replica 3 10.30.1.16:/var/lib/heketi/mounts/vg_9534f15dd9f0822ad454140d13c660a5/brick_ba4091b858d94a088b21a582d8d4abaa/brick 10.30.1.17:/var/lib/heketi/mounts/vg_a146220fd3f761e8da2be784523ce07e/brick_6f0ce82692e70ce5ae2ec55a60f237c6/brick 10.30.1.15:/var/lib/heketi/mounts/vg_19584b16bc8f21b87662b27b551652fb/brick_abcb32853351840ee82a95693cbb63b4/brick
Result: volume create: heketidbstorage: success: please start the volume to access data
```

进入任意GlusterFS Pod查看卷信息：

```bash
$ kubectl exec glusterfs-25cm8 -it bash
[root@ubuntu16 /]# gluster volume info

Volume Name: heketidbstorage
Type: Replicate
Volume ID: c8da2a4a-3066-4708-a59d-201d22decd92
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 10.30.1.16:/var/lib/heketi/mounts/vg_9534f15dd9f0822ad454140d13c660a5/brick_ba4091b858d94a088b21a582d8d4abaa/brick
Brick2: 10.30.1.17:/var/lib/heketi/mounts/vg_a146220fd3f761e8da2be784523ce07e/brick_6f0ce82692e70ce5ae2ec55a60f237c6/brick
Brick3: 10.30.1.15:/var/lib/heketi/mounts/vg_19584b16bc8f21b87662b27b551652fb/brick_abcb32853351840ee82a95693cbb63b4/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
[root@ubuntu16 /]# 
```

分析下heketi-storage.json：

```bash
#将创建如下资源信息：
Endpoints：
    name：heketi-storage-endpoints
         10.30.1.16:1 10.30.1.15:1 10.30.1.17:1
Service：    
    name: heketi-storage-endpoints  
job： 
   name: heketi-storage-copy-job
   images: heketi/heketi:dev
   声明了 volume：heketi-storage 
         "volumes": [
                                {
                                    "name": "heketi-storage",
                                    "glusterfs": {
                                        "endpoints": "heketi-storage-endpoints",
                                        "path": "heketidbstorage"
                                    }
                                },
     挂载到 /heketi：
             volumeMounts": [
                                            {
                                                "name": "heketi-storage",
                                                "mountPath": "/heketi"
                                            },

   启动时执行命令：cp /db/heketi.db /heketi 
   #由此可知，此job的作用就是复制heketi中的数据文件到 /heketi，而/heketi目录挂载在了卷heketi-storage中，而heketi-storage volume是前面执行"heketi-cli setup-openshift-heketi-storage"时创建好了的
```

创建之：

```bash
$ kubectl create -f heketi-storage.json
secret "heketi-storage-secret" created
endpoints "heketi-storage-endpoints" created
service "heketi-storage-endpoints" created
job "heketi-storage-copy-job" created
```

当Job执行完后就可以删除它了：

```bash
$ kubectl get job 
NAME                      DESIRED   SUCCESSFUL   AGE
heketi-storage-copy-job   1         1            1m
```

等到job完成后，删除bootstrap Heketi实例相关的组件：

```bash
#把之前由heketi-bootstrap.json创建的资源删除
$ kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi"
deployment "deploy-heketi" deleted
job "heketi-storage-copy-job" deleted
pod "deploy-heketi-69bfbd4bbd-q8tsk" deleted
service "deploy-heketi" deleted
secret "heketi-storage-secret" deleted
```

之前创建的名为deploy-heketi的pod,service已经删除了：

```bash
$ kubectl get pod
NAME              READY     STATUS              RESTARTS   AGE
glusterfs-25cm8   1/1       Running             1          1h
glusterfs-8qrpt   1/1       Running             1          1h
glusterfs-c4859   1/1       Running             1          1h

$ kubectl get svc
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
heketi-storage-endpoints      ClusterIP   10.254.191.233   <none>        1/TCP      4m
```

最后，使用heketi-deployment.json文件重新部署heketi

```bash
$ cat heketi-deployment.json 
{
  "kind": "List",
  "apiVersion": "v1",
  "items": [
    {
      "kind": "Secret",
      "apiVersion": "v1",
      "metadata": {
        "name": "heketi-db-backup",
        "labels": {
          "glusterfs": "heketi-db",
          "heketi": "db"
        }
      },
      "data": {
      },
      "type": "Opaque"
    },
    {
      "kind": "Service",
      "apiVersion": "v1",
      "metadata": {
        "name": "heketi",
        "labels": {
          "glusterfs": "heketi-service",
          "deploy-heketi": "support"
        },
        "annotations": {
          "description": "Exposes Heketi Service"
        }
      },
      "spec": {
        "selector": {
          "name": "heketi"
        },
        "ports": [
          {
            "name": "heketi",
            "port": 8080,
            "targetPort": 8080
          }
        ]
      }
    },
    {
      "kind": "Deployment",
      "apiVersion": "extensions/v1beta1",
      "metadata": {
        "name": "heketi",
        "labels": {
          "glusterfs": "heketi-deployment"
        },
        "annotations": {
          "description": "Defines how to deploy Heketi"
        }
      },
      "spec": {
        "replicas": 1,
        "template": {
          "metadata": {
            "name": "heketi",
            "labels": {
              "name": "heketi",
              "glusterfs": "heketi-pod"
            }
          },
          "spec": {
            "serviceAccountName": "heketi-service-account",
            "containers": [
              {
                "image": "heketi/heketi:dev",
                "imagePullPolicy": "Always",
                "name": "heketi",
                "env": [
                  {
                    "name": "HEKETI_EXECUTOR",
                    "value": "kubernetes"
                  },
                  {
                    "name": "HEKETI_DB_PATH",
                    "value": "/var/lib/heketi/heketi.db"
                  },
                  {
                    "name": "HEKETI_FSTAB",
                    "value": "/var/lib/heketi/fstab"
                  },
                  {
                    "name": "HEKETI_SNAPSHOT_LIMIT",
                    "value": "14"
                  },
                  {
                    "name": "HEKETI_KUBE_GLUSTER_DAEMONSET",
                    "value": "y"
                  }
                ],
                "ports": [
                  {
                    "containerPort": 8080
                  }
                ],
                "volumeMounts": [
                  {
                    "mountPath": "/backupdb",
                    "name": "heketi-db-secret"
                  },
                  {
                    "name": "db",
                    "mountPath": "/var/lib/heketi"
                  },
                  {
                    "name": "config",
                    "mountPath": "/etc/heketi"
                  }
                ],
                "readinessProbe": {
                  "timeoutSeconds": 3,
                  "initialDelaySeconds": 3,
                  "httpGet": {
                    "path": "/hello",
                    "port": 8080
                  }
                },
                "livenessProbe": {
                  "timeoutSeconds": 3,
                  "initialDelaySeconds": 30,
                  "httpGet": {
                    "path": "/hello",
                    "port": 8080
                  }
                }
              }
            ],
            "volumes": [
              {
                "name": "db",
                "glusterfs": {
                  "endpoints": "heketi-storage-endpoints",
                  "path": "heketidbstorage"
                }
              },
              {
                "name": "heketi-db-secret",
                "secret": {
                  "secretName": "heketi-db-backup"
                }
              },
              {
                "name": "config",
                "secret": {
                  "secretName": "heketi-config-secret"
                }
              }
            ]
          }
        }
      }
    }
  ]
}
```

heketi-deployment.json 文件 创建了如下资源：

```bash
Service:
     name: heketi
     port: 8080
Deployment：
    name: heketi
    replicas: 1
    image: heketi/heketi:dev
    volumeMounts: 
           name: db   mountPath: /var/lib/heketi 
   volumes: 
      endpoints: heketi-storage-endpoints   #由heketi-storage.json文件创建
       path：heketidbstorage  #这是gluster volume名，此volume是由"heketi-cli setup-openshift-heketi-storage"自动创建。
    # 将heketi容器内的/var/lib/heketi 目录挂载到了GlusterFS volume “heketidbstorage”中。
```

部署之：

```bash
$ kubectl create -f heketi-deployment.json 
secret "heketi-db-backup" created
service "heketi" created
deployment "heketi" created

$ kubectl get deployment 
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
heketi    1         1         1            1           45s
$ kubectl get svc 
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
heketi                        ClusterIP   10.254.239.189   <none>        8080/TCP   51s
heketi-storage-endpoints      ClusterIP   10.254.191.233   <none>        1/TCP      31m
```

验证heketi是否在用在用gluster volume：

```bash
$ kubectl get pod
NAME                      READY     STATUS              RESTARTS   AGE
glusterfs-25cm8           1/1       Running             1          1h
glusterfs-8qrpt           1/1       Running             1          1h
glusterfs-c4859           1/1       Running             1          1h
heketi-7898db85dd-nb6kn   1/1       Running             0          1m

$ kubectl exec heketi-7898db85dd-nb6kn -it bash
[root@heketi-7898db85dd-nb6kn /]# mount |grep heketi
10.30.1.15:heketidbstorage on /var/lib/heketi type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
```

至此，heketi db已正确配置了GlusterFS卷。

