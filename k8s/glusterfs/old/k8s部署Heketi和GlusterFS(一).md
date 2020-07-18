https://blog.51cto.com/newfly/2134514

## 前言

在kubernetes中，使用GlusterFS文件系统，操作步骤通常是：

**创建brick-->创建volume-->创建PV-->创建PVC-->Pod挂载PVC**

如果要创建多个PV，则需要手动重复执行这些繁锁步骤，那么使用GlusterFS如何动态创建呢，这就要借助于第三方的Heketi了，为什么要借助于Heketi呢，因为GlusterFS自身并不具备RESTful API，而k8s必须通过Restful API请求来创建pv。
Kubernetes可以通过Heketi管理GlusterFS卷的生命周期的，Heketi为GlusterFS提供了一个RESTful API 接口供Kubernetes调用，通过Heketi，Kubernetes可以动态配置GlusterFS卷，Heketi会动态在集群内选择bricks创建所需的volumes，确保数据的副本会分散到集群不同的故障域内，同时Heketi还支持GlusterFS多集群管理，便于管理员对GlusterFS进行操作。
Heketi要求在每个glusterfs节点上配备裸磁盘，因为Heketi要用来创建PV和VG，如果有了Heketi，则可以通过StorageClass来创建PV，步骤仅有：

**创建StorageClass-->创建PVC-->Pod挂载PVC。**

这种方式称为基于StorageClass的动态资源供应，虽然只有简单的两步，但是它所干活的活一点也不比上述中步骤少，只不过大部分工作都由Heketi在背后帮我们完成了。
本文的操作步骤依据[heketi的github网址官方文档](https://github.com/heketi/heketi/blob/master/docs/admin/install-kubernetes.md)。
本实例用到的所有文件都位于[extras/kubernetes](https://github.com/heketi/heketi/tree/master/extras/kubernetes)，在下载的heketi客户端工具包中也有示例文件。

本文的环境是在三个Kubernetes Node上部署三个GluserFS节点。

| 服务器 | 主机名称 | IP         | Storage IP | 磁盘     | 角色                    |
| ------ | -------- | ---------- | ---------- | -------- | ----------------------- |
| Node1  | ubuntu15 | 10.30.1.15 | 10.30.1.15 | /dev/sdb | K8s Node+GlusterFS Node |
| Node2  | ubuntu16 | 10.30.1.16 | 10.30.1.16 | /dev/sdb | K8s Node+GlusterFS Node |
| Node3  | ubuntu17 | 10.30.1.17 | 10.30.1.17 | /dev/sdb | K8s Node+GlusterFS Node |

> 注意：
> Heketi要至少需要三个GlusterFS节点。
> 加载内核模块：每个kubernetes集群的节点运行**modprobe dm_thin_pool**。

## 下载heketi客户端工具

Heketi提供了一个CLI，方便用户在Kubernetes中管理和配置GlusterFS,在客户端机器上[下载heketi客户端](https://github.com/heketi/heketi/releases)工具到合适的位置，版本必须与heketi server的版本一致。

## 在集群内部署glusterfs-server

#### glusterfs以DaemonSet方式部署，几乎不用做修改，但是还是要了解这个文件做了什么：

```bash
      glusterfs-daemonset.json  
      资源类型：daemonset
         nodeSelector： 
                  storagenode: glusterfs
         name: glusterfs
         image: gluster/gluster-centos:latest  
         "livenessProbe": 
                   command: /bin/bash -c systemctl status glusterd.service
                   timeoutSeconds: 3
                  "initialDelaySeconds": 60,
         volumeMounts:
               宿主机目录---->容器内目录 
               /var/lib/heketi---> /var/lib/heketi
               /run/lvm ---> "/run/lvm
               /run----/run
               /etc/glusterfs---->/etc/glusterfs
               /var/log/glusterfs---->/var/log/glusterfs
               /var/lib/glusterd--->"/var/lib/glusterd
               /dev---->/dev
               /sys/fs/cgroup---->/sys/fs/cgroug
```

#### 给需要部署GlusterFS节点的Node打上标签

```bash
$root@ubuntu15:~# kubectl label node 10.30.1.15 storagenode=glusterfs
    node "10.30.1.15" labeled
$root@ubuntu15:~# kubectl label node 10.30.1.16 storagenode=glusterfs
    node "10.30.1.16" labeled
$#root@ubuntu15:~# kubectl label node 10.30.1.17 storagenode=glusterfs
    node "10.30.1.17" labeled
```

#### 部署并验证

```bash
 $root@ubuntu15:#  kubectl create -f glusterfs-daemonset.json 
    daemonset "glusterfs" created

 $root@ubuntu15:#  kubectl get pod
    NAME                             READY     STATUS    RESTARTS   AGE
    glusterfs-94g22                  1/1       Running   0          2m
    glusterfs-bc8tb                  1/1       Running   0          2m
    glusterfs-n22c8                  1/1       Running   0          2m
```

## 在集群内部署heketi服务端

了解部署文件做了什么：heketi-bootstrap.json：

```bash
       deployment " deploy-heketi"
          image: heketi/heketi:dev
          name": deploy-heketi
          containerPort: 8080
          serviceAccountName: heketi-service-account
          secretName： heketi-config-secret
        Service: 
            name: deploy-heketi
            port: 8080
            targetport: 8080
```

#### 根据deploy文件为Heketi创建对应的服务帐户：

```bash
 root@ubuntu15:# kubectl create -f heketi-service-account.json
 serviceaccount "heketi-service-account" created
```

#### 为服务帐户创建集群角色绑定，以授权控制gluster的pod

```bash
$ kubectl create clusterrolebinding heketi-gluster-admin --clusterrole=edit --serviceaccount=default:heketi-service-account
```

> 此处授权的名称空间为default，意味着，Heketi所能操作的gluster-server之Pod也在此名称空间内，否则此角色将无法访问到gluster-server。

#### 创建secret来保存Heketi服务的配置

```bash
root@ubuntu15:$ kubectl create secret generic heketi-config-secret --from-file=./heketi.json
secrets "heketi-config-secret" created
```

> 1.必须将配置文件heketi.json中的glusterfs/executor设置为kubernetes，如此才能让Heketi服务控制GlusterFS Pod。
> 2.Secret的名称空间，必须与gluserfs Pod位于同一名称空间内才能挂载之。

#### 部署并验证一切正常运行：

```bash
$ kubectl create -f heketi-bootstrap.json
  service "deploy-heketi" created
  deployment "deploy-heketi" created

$ kubectl get pod
     NAME                             READY     STATUS    RESTARTS   AGE
     deploy-heketi-8465f8ff78-sb8z   1/1       Running   0          3m
     glusterfs-94g22                  1/1       Running   0          28m
     glusterfs-bc8tb                   1/1       Running   0          28m
     glusterfs-n22c8                  1/1       Running   0          28m
```

#### 测试Heketi服务端

既然Bootstrap Heketi服务正在运行，我们将配置端口转发，以便我们可以使用Heketi CLI与服务端进行通信。使用Heketi pod的名称，运行下面的命令：

```bash
$ kubectl port-forward  deploy-heketi-8465f8ff78-sb8z 8080:8080
$ curl http://localhost:8080/hello
   Handling connection for 8080
   Hello from heketi
```

## Heketi管理gluster-server

```bash
#首先确定glusterfs 及heketi服务端pod的运行情况：
    root@ubuntu15:$ kubectl get pod 
    NAME                             READY     STATUS    RESTARTS   AGE
    deploy-heketi-8465f8ff78-sb8zv   1/1       Running   0          20m
    glusterfs-6pf8q                  1/1       Running   0          45m
    glusterfs-kn6jf                  1/1       Running   9           45m
    glusterfs-m2jt4                  1/1       Running   0          45m
    root@ubuntu15:$ kubectl get svc
    NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    deploy-heketi             ClusterIP   10.254.238.186      <none>             8080/TCP   1h
```

使用Heketi-cli命令行工具向Heketi提供要管理的GlusterFS集群的信息。但是现在它并不知道自已的服务装是哪个,它通过变量**HEKETI_CLI_SERVER**来找自已的服务端，因此设置变量：

```bash
export HEKETI_CLI_SERVER=http://10.254.238.186:8080  #heketi的service的Cluster IP及Port
```

在示例文件中有个topology-sample.json文件，称为拓朴文件，它提供了运行gluster Pod的kubernetes节点IP，每个节点上相应的磁盘块设备，修改hostnames/manage，设置为与kubectl get nodes所显示的Name字段的值，通常为Node IP，修改hostnames/storage下的IP，为存储网络的IP地址，也即Node IP。

```bash
$ cat topology-sample.json 
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.30.1.15"
              ],
              "storage": [
                "10.30.1.15"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.30.1.16"
              ],
              "storage": [
                "10.30.1.16"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.30.1.17"
              ],
              "storage": [
                "10.30.1.17"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```

然后加载就可以了：

```bash
$ heketi-cli topology load --json=topology-sample.json  
   Creating cluster ... ID: 224a5a6555fa5c0c930691111c63e863
     Allowing file volumes on cluster.
     Allowing block volumes on cluster.
     Creating node 10.30.1.15 ... ID: 7946b917b91a579c619ba51d9129aeb0
            Adding device /dev/sdb ... OK
     Creating node 10.30.1.16 ... ID: 5d10e593e89c7c61f8712964387f959c
            Adding device /dev/sdb ... OK
     Creating node 10.30.1.17 ... ID: de620cb2c313a5461d5e0a6ae234c553
            Adding device /dev/sdb ... OK
```

> ** 注意：必须使用与服务器版本匹配的Heketi-cli版本加载拓扑文件。

那么 在执行了heketi-cli topology load之后，Heketi到底在服务器上做了什么呢？

1. 进入任意glusterfs Pod内，执行gluster peer status 发现都已把对端加入到了可信存储池(TSP)中。
2. 在运行了gluster Pod的节点上，自动创建了一个VG，此VG正是由topology-sample.json 文件中的磁盘裸设备创建而来。
3. 一块磁盘设备创建出一个VG，以后创建的PVC，即从此VG里划分的LV。
4. heketi-cli topology info 查看拓扑结构，显示出每个磁盘设备的ID，对应VG的ID，总空间、已用空间、空余空间等信息。
5. 这一切操作都可以通过Heketi Pod 日志得到证实：

```bash
$ kubectl log -f deploy-heketi-8465f8ff78-sb8z 
# 只截取部分日志
[heketi] INFO 2018/06/29 15:05:52 Adding node 10.30.1.15
[heketi] INFO 2018/06/29 15:05:52 Adding device /dev/sdb to node 18792ee65da0463eafab7281e0def378  
[negroni] Completed 202 Accepted in 1.587583ms

[kubeexec] DEBUG 2018/06/29 15:05:52 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: pvcreate --metadatasize=128M --dataalignment=256K '/dev/sdb'
Result:   Physical volume "/dev/sdb" successfully created.
[kubeexec] DEBUG 2018/06/29 15:05:53 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: vgcreate --autobackup=n vg_06a31aebc9e80ff7a53908942e82236d /dev/sdb
Result:   Volume group "vg_06a31aebc9e80ff7a53908942e82236d" successfully created
[kubeexec] DEBUG 2018/06/29 15:05:53 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: vgdisplay -c vg_06a31aebc9e80ff7a53908942e82236d
Result:   vg_06a31aebc9e80ff7a53908942e82236d:r/w:772:-1:0:0:0:-1:0:1:1:20836352:4096:5087:0:5087:IWxRep-wsIT-pJuy-PfgW-E5d1-GodE-sZeVet
[cmdexec] DEBUG 2018/06/29 15:05:53 /src/github.com/heketi/heketi/executors/cmdexec/device.go:147: Size of /dev/sdb in 10.30.1.15 is 20836352
[heketi] INFO 2018/06/29 15:05:53 Added device /dev/sdb
[asynchttp] INFO 2018/06/29 15:05:53 asynchttp.go:292: Completed job 700b875feeeaf8818d16967dd18b8c3a in 583.847611ms

[heketi] INFO 2018/06/29 15:05:53 Adding node 10.30.1.16
[negroni] Completed 202 Accepted in 86.946338ms
[asynchttp] INFO 2018/06/29 15:05:53 asynchttp.go:288: Started job 8f5da3c1261253d1ce80296553093e96
[cmdexec] INFO 2018/06/29 15:05:53 Probing: 10.30.1.15 -> 10.30.1.16
[negroni] Started GET /queue/8f5da3c1261253d1ce80296553093e96
[negroni] Completed 200 OK in 39.252µs
[negroni] Started GET /queue/8f5da3c1261253d1ce80296553093e96
[negroni] Completed 200 OK in 64.031µs
[kubeexec] DEBUG 2018/06/29 15:05:53 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: gluster peer probe 10.30.1.16
Result: peer probe: success. Host 10.30.1.16 port 24007 already in peer list
[cmdexec] INFO 2018/06/29 15:05:53 Setting snapshot limit
[kubeexec] DEBUG 2018/06/29 15:05:54 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: gluster --mode=script snapshot config snap-max-hard-limit 14
Result: snapshot config: snap-max-hard-limit for System set successfully
[heketi] INFO 2018/06/29 15:05:54 Added node 7420ad8b19098c806117df6b726686dd
[asynchttp] INFO 2018/06/29 15:05:54 asynchttp.go:292: Completed job 8f5da3c1261253d1ce80296553093e96 in 443.362246ms

[heketi] INFO 2018/06/29 15:05:54 Adding device /dev/sdb to node 7420ad8b19098c806117df6b726686dd

[kubeexec] DEBUG 2018/06/29 15:05:54 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-n22c8 Command: pvcreate --metadatasize=128M --dataalignment=256K '/dev/sdb'
Result:   Physical volume "/dev/sdb" successfully created.
[kubeexec] DEBUG 2018/06/29 15:05:55 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-n22c8 Command: vgcreate --autobackup=n vg_e8b4af1aca6de676042ec273e34cf1d6 /dev/sdb
Result:   Volume group "vg_e8b4af1aca6de676042ec273e34cf1d6" successfully created
[kubeexec] DEBUG 2018/06/29 15:05:55 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.16 Pod: glusterfs-n22c8 Command: vgdisplay -c vg_e8b4af1aca6de676042ec273e34cf1d6
Result:   vg_e8b4af1aca6de676042ec273e34cf1d6:r/w:772:-1:0:0:0:-1:0:1:1:20836352:4096:5087:0:5087:tlpvcR-6720-nUc8-xKcn-6Ga3-pufv-YOu1NA
[cmdexec] DEBUG 2018/06/29 15:05:55 /src/github.com/heketi/heketi/executors/cmdexec/device.go:147: Size of /dev/sdb in 10.30.1.16 is 20836352
[heketi] INFO 2018/06/29 15:05:55 Added device /dev/sdb
[asynchttp] INFO 2018/06/29 15:05:55 asynchttp.go:292: Completed job 768f5d4d7bccb9366b12ca38c0fd762d in 958.352618ms

[cmdexec] INFO 2018/06/29 15:05:55 Check Glusterd service status in node 10.30.1.15
[kubeexec] DEBUG 2018/06/29 15:05:55 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb 
[heketi] INFO 2018/06/29 15:05:55 Adding node 10.30.1.17
[negroni] Completed 202 Accepted in 80.15039ms
[asynchttp] INFO 2018/06/29 15:05:55 asynchttp.go:288: Started job 5f5ddb77130bf672f82c370d3a33e7fb
[cmdexec] INFO 2018/06/29 15:05:55 Probing: 10.30.1.15 -> 10.30.1.17
[negroni] Started GET /queue/5f5ddb77130bf672f82c370d3a33e7fb
[kubeexec] DEBUG 2018/06/29 15:05:56 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: gluster peer probe 10.30.1.17
Result: peer probe: success. 

[kubeexec] DEBUG 2018/06/29 15:05:56 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.15 Pod: glusterfs-bc8tb Command: gluster --mode=script snapshot config snap-max-hard-limit 14
Result: snapshot config: snap-max-hard-limit for System set successfully
[heketi] INFO 2018/06/29 15:05:56 Added node e0e240d4dede978f38b7ccc82e218d11
[asynchttp] INFO 2018/06/29 15:05:56 asynchttp.go:292: Completed job 5f5ddb77130bf672f82c370d3a33e7fb in 1.023782431s

[negroni] Started POST /devices
[heketi] INFO 2018/06/29 15:05:56 Adding device /dev/sdb to node e0e240d4dede978f38b7ccc82e218d11
[negroni] Completed 202 Accepted in 1.587062ms

[kubeexec] DEBUG 2018/06/29 15:05:58 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-94g22 Command: pvcreate --metadatasize=128M --dataalignment=256K '/dev/sdb'
Result:   Physical volume "/dev/sdb" successfully created.
[kubeexec] DEBUG 2018/06/29 15:05:58 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-94g22 Command: vgcreate --autobackup=n vg_e32a3d835afdfefec890ee91edb6fe57 /dev/sdb
Result:   Volume group "vg_e32a3d835afdfefec890ee91edb6fe57" successfully created
[kubeexec] DEBUG 2018/06/29 15:05:58 /src/github.com/heketi/heketi/executors/kubeexec/kubeexec.go:246: Host: 10.30.1.17 Pod: glusterfs-94g22 Command: vgdisplay -c vg_e32a3d835afdfefec890ee91edb6fe57
Result:   vg_e32a3d835afdfefec890ee91edb6fe57:r/w:772:-1:0:0:0:-1:0:1:1:20836352:4096:5087:0:5087:gcBVHV-5Iw9-fvz9-Q07N-Kq3e-ahwM-efVef7
[cmdexec] DEBUG 2018/06/29 15:05:58 /src/github.com/heketi/heketi/executors/cmdexec/device.go:147: Size of /dev/sdb in 10.30.1.17 is 20836352
[heketi] INFO 2018/06/29 15:05:58 Added device /dev/sdb
```

## Heketi管理GlusterFS的简单示例。

heketi服务部署完成了，现在通过简单示例来说明如何使用，有两种方法来调配存储。常用的方法是设置一个StorageClass，让Kubernetes为提交的PersistentVolumeClaim自动配置存储。或者，可以通过Kubernetes手动创建和管理卷（PV），或直接使用heketi-cli中的卷。
下面的用法示例参考[gluster-kubernetes hello world示例](https://github.com/gluster/gluster-kubernetes/blob/master/docs/examples/hello_world/README.md)

#### 创建一个StorageClass

```bash
 $ cat gluster-storage-class.yaml 
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gluster-heketi                        #-----存储类的名字
    provisioner: kubernetes.io/glusterfs
    parameters:
      resturl: "http://10.254.238.186:8080"      #------heketi service的cluster ip 和端口
      restuser: "admin"                    #--heketi的认证用户，这里随便填，因为没有启用鉴权模式 
      gidMin: "40000"
      gidMax: "50000"
      volumetype: "replicate:3"            #--申请的默认为3副本模式，因为目前有三个gluster节点。
```

#### 创建一个pvc

```bash
$cat gluster-pvc.yaml 
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: gluster1
      annotations:
        volume.beta.kubernetes.io/storage-class: gluster-heketi    #---上面创建的存储类的名称
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 2Gi
```

PVC的定义一旦生成，系统便会触发发Heketi进行相应的操作，主要是为GlusterFS创建brick及volume，查看pvc：

```bash
# 自动绑定
$ kubectl get pvc
NAME              STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
gluster1          Bound     pvc-6784c33b-7acb-11e8-bdec-000c29774d39   2G         RWX            gluster-heketi   6m
```

创建pvc后查看服务器上发生的变化：

```bash
$ vgs
  VG                                  #PV #LV #SN Attr   VSize  VFree 
  vg_06a31aebc9e80ff7a53908942e82236d   1   1   0 wz--n- 19.87g 18.83g

$  lvs
  LV                       VG               Attr      LSize  Pool                                Origin Data%  Move Log Copy%  Convert   
    brick_c2e5e57f2574bec14c8821ef3e163d2a vg_06a31aebc9e80ff7a53908942e82236d Vwi-aotz-  2.00g tp_c2e5e57f2574bec14c8821ef3e163d2a          0.70   
```

**由此可见：一个pvc对应一个brick，一个brick对应一个LV。**

#### 部署一个nginx Pod来挂载pvc

```bash
$ cat heketi-nginx.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod1
  labels:
    name: nginx-pod1
spec:
  containers:
  - name: nginx-pod1
    image: gcr.io/google_containers/nginx-slim:0.8
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: gluster-vol1
      mountPath: /usr/share/nginx/html
  volumes:
  - name: gluster-vol1
    persistentVolumeClaim:
      claimName: gluster1   #上面创建的pvc

$ kubectl create -f nginx-pod.yaml
   pod "nginx-pod1" created

#查看Pod
$ kubectl get pod -o wide
        NAME                             READY     STATUS    RESTARTS   AGE       IP                NODE
        deploy-heketi-8465f8ff78-sb8z   1/1       Running   0          39m       192.168.150.218   10.30.1.16
        glusterfs-94g22                  1/1       Running   1          1h       10.30.1.17         10.30.1.17
        glusterfs-bc8tb                  1/1       Running   2          1h       10.30.1.15         10.30.1.15
        glusterfs-n22c8                  1/1       Running   3         1h       10.30.1.16         10.30.1.16
        nginx-pod1                       1/1       Running   0            2m        192.168.47.207    10.30.1.15
```

验证：
现在到nginx容器中创建一个index.html文件

```bash
$ kubectl exec -it nginx-pod1 /bin/sh
# cd /usr/share/nginx/html
# echo 'Hello World from GlusterFS!!!' > index.html
# ls
index.html
# exit
```

测试运行的nginx Pod正常性：

```bash
$ curl http://192.168.47.207
Hello World from GlusterFS!!!
```

现在进入到gluster Pod 三个Pod中的任意一个Pod都行， 看看刚刚创建的index.html文件:

```bash
# 进入到10.30.1.15上的pod查看，先查看在10.30.1.15的vg名称：vg_c88262b05d49d3ef1b94a31636a549a7  进入到Pod 查看此VG的挂载点：
[root@ubuntu15 /]# mount |grep vg_c8826
/dev/mapper/vg_c88262b05d49d3ef1b94a31636a549a7-brick_451f81bc629344f71fab63a30fab1773 on /var/lib/heketi/mounts/vg_c88262b05d49d3ef1b94a31636a549a7/brick_451f81bc629344f71fab63a30fab1773 type xfs (rw,noatime,nouuid,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota)

# 根据它的挂载位置，cd到挂载目录，查看创建的文件：
[root@ubuntu15 brick]# pwd /var/lib/heketi/mounts/vg_c88262b05d49d3ef1b94a31636a549a7/brick_451f81bc629344f71fab63a30fab1773/brick

[root@ubuntu15 brick]# cat index.html 
Hello World from GlusterFS!!! 
```

通过gluster volume info查看到volume类型为Replicate，并且有三个副本，因此在三个gluster Pod中对应挂载点都会看到此文件。

**此文仅用于理解Heketi如何动态管理GluseterFS来完成动态资源供应，还不能直接用于生产环境 。**

