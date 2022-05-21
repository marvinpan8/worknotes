## 注意事项

- 安装Glusterfs客户端：每个kubernetes集群的节点需要安装gulsterfs的客户端，
```bash
yum -y install centos-release-gluster41.noarch
yum -y install glusterfs-client
```
- 加载内核模块：每个kubernetes集群的节点运行
```bash
modprobe dm_thin_pool
```
- 至少三个slave节点：至少需要3个kubernetes slave节点用来部署glusterfs集群，并且这3个slave节点每个节点需要至少一个空余的磁盘 。

  ```bash
  file -s /dev/sdc
  -------------------
  #显示为空盘，若不是，参照我的glusterfs部署-删除挂载
  /dev/sdc: data
  ```

  

#### Glusterfs和Heketi在Kubernetes集群中的部署过程

- 通过设置storagenode=glusterfs节点上的标签，将gluster容器部署到指定节点上。

```bash
$ kubectl label node 192.168.10.113 storagenode=glusterfs
$ kubectl label node 192.168.10.122 storagenode=glusterfs
$ kubectl label node 192.168.10.123 storagenode=glusterfs
----
$ kubectl taint nodes 192.168.10.113 storagenode=glusterfs:NoSchedule
$ kubectl taint nodes 192.168.10.122 storagenode=glusterfs:NoSchedule
$ kubectl taint nodes 192.168.10.123 storagenode=glusterfs:NoSchedule
----
$ kubectl taint nodes 192.168.10.114 node=harbor:NoSchedule
$ kubectl taint nodes 192.168.10.125 node=harbor:NoSchedule
```

- glusterfs-daemonset.json 加上容忍

```bash
"tolerations": [
  {
    "key": "storagenode"
    "operator": "Equal"
    "value": "glusterfs"
    "effect": "NoSchedule"
  }
]
```

- 部署 GlusterFS DaemonSet

```bash
$ kubectl create -f glusterfs-daemonset.json
```

- 验证Pod在节点上运行至少应运行3个Pod（因此至少需要给3个节点打标签）。

```bash
$ kubectl get po -o wide
```

- 接下来，我们将为Heketi创建一个服务帐户（service-account）:

```bash
$ kubectl create -f heketi-service-account.json
```

- 我们现在必须给该服务帐户的授权绑定相应的权限来控制gluster的pod。我们通过为我们新创建的服务帐户创建群集角色绑定（cluster role binding）来完成此操作。

```bash
$ kubectl create clusterrolebinding heketi-gluster-admin --clusterrole=edit --serviceaccount=glusterfs:heketi-service-account
```

- 现在我们需要创建一个Kubernetes secret来保存我们Heketi实例的配置。必须将配置文件的执行程序设置为 kubernetes才能让Heketi server控制gluster pod（配置文件的默认配置）。除此这些，可以尝试配置的其他选项。

```bash
$ kubectl create secret generic heketi-config-secret --from-file=./heketi.json
```

- 接下来，我们需要部署一个初始（bootstrap）Pod和一个服务来访问该Pod。在你用git克隆的repo中，会有一个heketi-bootstrap.json文件。
- 修改 heketi-bootstrap.json
```bash
"nodeSelector": {
    "node-role.kubernetes.io/harbor": "harbor"
},
"tolerations": [
    {
        "key": "node",
        "operator": "Equal",
        "value": "harbor",
        "effect": "NoSchedule"
    }
],
```
- 提交文件并验证一切正常运行，如下所示：

```bash
$ kubectl create -f heketi-bootstrap.json
service "deploy-heketi" created
deployment "deploy-heketi" created

$ kubectl get po -o wide
NAME                                                      READY     STATUS    RESTARTS   AGE
deploy-heketi-1211581626-2jotm                            1/1       Running   0          35m
glusterfs-ip-172-20-0-217.ec2.internal-1217067810-4gsvx   1/1       Running   0          1h
glusterfs-ip-172-20-0-218.ec2.internal-2001140516-i9dw9   1/1       Running   0          1h
glusterfs-ip-172-20-0-219.ec2.internal-2785213222-q3hba   1/1       Running   0          1h
```

- 现在通过对Heketi服务运行示例查询来验证端口转发是否正常。该命令应该已经打印了将从其转发的本地端口。将其合并到URL中以测试服务，如下所示：

```bash
$ kubectl get svc
$ curl http://10.254.205.55:8080/hello
Hello from heketi
```

- 最后，为Heketi CLI客户端设置一个环境变量，以便它知道Heketi服务器的地址。

```bash
$ export HEKETI_CLI_SERVER=http://10.254.201.109:8080
```

- 接下来，我们将向Heketi提供有关要管理的GlusterFS集群的信息。通过拓扑文件提供这些信息。克隆的repo中有一个示例拓扑文件，名为topology-sample.json。拓扑指定运行GlusterFS容器的Kubernetes节点以及每个节点的相应原始块设备。

- 确保hostnames/manage指向如下所示的确切名称kubectl get nodes得到的主机名（如ubuntu-1），并且hostnames/storage是存储网络的IP地址（对应ubuntu-1的ip地址）。

- **IMPORTANT**: 重要提示，目前，必须使用与服务器版本匹配的Heketi-cli版本加载拓扑文件。另外，Heketi pod 带有可以通过`kubectl exec ...`访问的heketi-cli副本。

-  修改拓扑文件以反映您所做的选择，然后如下所示部署它（修改主机名，IP，block 设备的名称 如xvdg）：

```bash
$ heketi-cli --user admin --secret jrtz0755 topology load --json=topology-sample.json
----------------------------------
Handling connection for 57598
	Found node ip-172-20-0-217.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
	Found node ip-172-20-0-218.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
	Found node ip-172-20-0-219.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
```

- 接下来，我们将使用heketi为其存储其数据库提供一个卷（不要怀疑，就是使用这个命令，openshift和kubernetes通用，此命令生成heketi-storage.json文件）：
- [heketi命令网站](https://www.systutorials.com/docs/linux/man/8-heketi-cli/)

```bash
$ heketi-cli --user admin --secret jrtz0755 setup-openshift-heketi-storage
# 生成了 heketi-storage.json
$ kubectl create -f heketi-storage.json
secret/heketi-storage-secret created
endpoints/heketi-storage-endpoints created
service/heketi-storage-endpoints created
job.batch/heketi-storage-copy-job created
```

> Pitfall: 注意，如果在运行setup-openshift-heketi-storage子命令时heketi-cli报告“无空间”错误，则可能无意中运行topology load命令的时候服务端和heketi-cli的版本不匹配造成的。停止正在运行的heketi pod（kubectl scale deployment deploy-heketi –replicas=0），手动删除存储块设备中的任何签名，然后继续运行heketi pod（kubectl scale deployment deploy-heketi –replicas=1）。然后用匹配版本的heketi-cli重新加载拓扑，然后重试该步骤。

- 等到作业完成后，删除bootstrap Heketi实例相关的组件：

```bash
# 查看job完成 DURATION 秒数, po Completed
$ kubectl get jobs,po
NAME                      COMPLETIONS   DURATION   AGE
heketi-storage-copy-job   1/1           39s        2m55s
NAME                                 READY   STATUS      RESTARTS   AGE
pod/deploy-heketi-54b58c598f-f95q2   1/1     Running     0          23m
pod/glusterfs-2dpkq                  1/1     Running     0          38m
pod/glusterfs-9gj5q                  1/1     Running     0          38m
pod/glusterfs-svpc7                  1/1     Running     0          38m
pod/heketi-storage-copy-job-qp5x5    0/1     Completed   0          65s
# 删除所有deploy-heketi资源
$ kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi"
pod "deploy-heketi-54b58c598f-f95q2" deleted
service "deploy-heketi" deleted
deployment.apps "deploy-heketi" deleted
replicaset.apps "deploy-heketi-54b58c598f" deleted
job.batch "heketi-storage-copy-job" deleted
secret "heketi-storage-secret" deleted
```

- **heketi-deployment.json增加 nodeSelector，tolerations**
- 创建长期使用的Heketi实例（存储持久化的）：

```bash
$ kubectl create -f heketi-deployment.json
secret/heketi-db-backup created
service/heketi created
deployment.extensions/heketi created
```

**这样做了以后，heketi db将使用GlusterFS卷，并且每当heketi pod重新启动时都不会重置（数据不会丢失，存储持久化）。**

确认先前建立的集群存在，并且heketi可以列出在bootstrap阶段创建的db存储卷。

## 查看GlusterFS节点

**进入glusterfs容器查看**

```bash
# 以113容器为例
$ kubectl exec -it glusterfs-t9t4z bash
[root@vmsrv-010-113 /]$ lsblk
[root@vmsrv-010-113 /]$ df -Th
/dev/mapper/vg_3c161e1df90858773210ee093d7c2a18-brick_06528adcac48fcbf389a160fe7fd862e  2.0G   33M  2.0G   2% /var/lib/heketi/mounts/vg_3c161e1df90858773210ee093d7c2a18/brick_06528adcac48fcbf389a160fe7fd862e
---------------------------------------------------------------------
# 进入容器目录查看文件
[root@vmsrv-010-113 /]$ ls -l /var/lib/heketi/mounts/vg_3c161e1df90858773210ee093d7c2a18
/brick_06528adcac48fcbf389a160fe7fd862e/brick
----------------------
-rw-r--r-- 2 root root   371 Jul 19 11:52 container.log
-rw-r--r-- 2 root root 49152 Jul 19 11:52 heketi.db
-------------------------
# 查看集群状态
[root@vmsrv-010-113 /]$ gluster peer status
# 查看volume的具体信息：2副本的replicate卷；
# 另有 vgscan， vgdisplay 也可查看逻辑卷组信息等
[root@vmsrv-010-113 /]$ gluster volume list
Number of Peers: 2
Hostname: vmsrv-010-113
Uuid: 284c0207-a3c4-4891-8daf-5a1718ee3b60
State: Peer in Cluster (Connected)
Hostname: 192.168.10.143
Uuid: 25b6ed92-6355-425a-adc9-25202419bf9a
State: Peer in Cluster (Connected)
[root@vmsrv-010-113 /]$ gluster volume info vol_98f6395a2fb7c34a252666a3b0613835
Volume Name: vol_98f6395a2fb7c34a252666a3b0613835
Type: Replicate   # 可以看到 Type: Replicate复制卷
Volume ID: 684dff05-7792-4d71-8a97-3261c288a12e
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 192.168.10.113:/var/lib/heketi/mounts/vg_3c161e1df90858773210ee093d7c2a18/brick_9e7187459d819b14b49413b6317c0cc0/brick
Brick2: 192.168.10.122:/var/lib/heketi/mounts/vg_541670bc3c578e711aad2b92639008cb/brick_5a5574b65cb0b7dd8c367d835368b1e7/brick
Brick3: 192.168.10.123:/var/lib/heketi/mounts/vg_08be96cbb34a2d9ce4dc220fe1eb2659/brick_f7325cdea204a55c9c7f4d94d1704d3b/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

## 查看卷信息

进入124机器，有 heketi-cli 命令

```bash
$ export HEKETI_CLI_SERVER=http://10.254.201.109:8080
$ heketi-cli --user admin --secret jrtz0755 topology info
$ heketi-cli --user admin --secret jrtz0755 cluster list
$ heketi-cli --user admin --secret jrtz0755 node list
$ heketi-cli --user admin --secret jrtz0755 volume list
Id:ca43beff5901fec7f3cfd2cf2d35b861    Cluster:81941592b046cfdec2d416c4a8ec883e    Name:heketidbstorage
# 查看卷信息
$ heketi-cli --user admin --secret jrtz0755 volume info ca43beff5901fec7f3cfd2cf2d35b861
Name: heketidbstorage
Size: 2
Volume Id: ca43beff5901fec7f3cfd2cf2d35b861
Cluster Id: 81941592b046cfdec2d416c4a8ec883e
Mount: 192.168.10.123:heketidbstorage
Mount Options: backup-volfile-servers=192.168.10.122,192.168.10.113
Block: false
Free Size: 0
Reserved Size: 0
Block Hosting Restriction: (none)
Block Volumes: []
Durability Type: replicate
Distributed+Replica: 3
# 查看设备信息
$ heketi-cli --user admin --secret jrtz0755 device info d2e4abdb5f2140d77a4d16c4f994bbed
Name: /dev/sdb
State: online
Size (GiB): 499
Used (GiB): 3
Free (GiB): 496
Bricks:
Id:06528adcac48fcbf389a160fe7fd862e   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_3c161e1df90858773210ee093d7c2a18/brick_06528adcac48fcbf389a160fe7fd862e/brick
Id:9e7187459d819b14b49413b6317c0cc0   Size (GiB):1       Path: /var/lib/heketi/mounts/vg_3c161e1df90858773210ee093d7c2a18/brick_9e7187459d819b14b49413b6317c0cc0/brick
```



---

# 使用样例

有两种方法来调配存储。常用的方法是设置一个StorageClass，让Kubernetes为提交的PersistentVolumeClaim自动配置存储。或者，可以通过Kubernetes手动创建和管理卷（PVs），或直接使用heketi-cli中的卷。

参考[gluster-kubernetes hello world example](https://github.com/gluster/gluster-kubernetes/blob/master/docs/examples/hello_world/README.md) 获取关于 storageClass 的更多信息.

**使用PV与PVC绑定的注意项**：

- PV和PVC都受namespace的 限制，只有相同namespace中的PV和PVC才能绑定，并只有相同namespace下pod才能挂载PVC。
- 当在PVC定义中同时设置了selector和storageClassName，只有二者同时满足条件才能将PV和PVC绑定。

# 我的示例（非翻译部分内容）

确认glusterfs和heketi的pod运行正常

```bash
$ kubectl get pod 
glusterfs-f8tz6          1/1     Running   0          178m
glusterfs-rzs95          1/1     Running   0          178m
glusterfs-wfg75          1/1     Running   0          178m
heketi-7d55bbc85-tpcwj   1/1     Running   0          13m
```

创建密码secret

base64密码  # base64 encoded password. E.g.: echo -n "jrtz0755" | base64

```bash
cat <<EOF | tee heketi-secret-password.yml
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: glusterfs
data:
  key: anJ0ejA3NTU=
type: kubernetes.io/glusterfs
EOF
```

#### StorageClass yaml文件示例

```yaml
cat <<EOF | tee storage-class-default.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
  namespace: glusterfs
provisioner: kubernetes.io/glusterfs
allowVolumeExpansion: true
parameters:
  resturl: "http://10.254.39.223:8080"
  restuser: "admin"
  secretNamespace: "glusterfs"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
EOF
```
- allowVolumeExpansion: true------允许扩容

- resturl---heketi service的cluster ip 和端口

- restuser---在 Gluster 可信池中有权创建卷的 Gluster REST服务/Heketi 用户

- volumetype---replicate:3---申请的默认为3副本模式

- `gidMin`，`gidMax`：storage class GID 范围的最小值和最大值。在此范围（gidMin-gidMax）内的唯一值（GID）将用于动态分配卷。 这些是可选的值。如果不指定，卷将被分配一个 2000-2147483647 之间的值，这是 gidMin 和 gidMax 的默认值。

- `clusterid`：`630372ccdc720a92c681fb928f27b53f` 是集群的 ID，当分配卷时，Heketi 将会使用这个文件。 它也可以是一个 clusterid 列表，例如： `"8452344e2becec931ece4e33c4674e4e,42982310de6c63381718ccfa6d8cf397"`。这个是可选参数。

- secretNamespace，secretName：Secret 实例的标识，包含与 Gluster REST 服务交互时使用的用户密码。 这些参数是可选的，secretNamespace 和 secretName都省略是使用空密码。提供的密码必须有 “kubernetes.io/glusterfs” type，例如以这种方式创建：

  ```bash
  $ kubectl create secret generic heketi-secret \
    --type="kubernetes.io/glusterfs" --from-literal=key='opensesame' \
    --namespace=glusterfs
  ```

  secret 都例子可以在 [glusterfs-provisioning-secret.yaml](https://github.com/kubernetes/examples/tree/master/staging/persistent-volume-provisioning/glusterfs/glusterfs-secret.yaml) 中找到。

> 当动态分配 persistent volume 时，Gluster 插件自动创建一个端点和一个 以 gluster-dynamic- <claimname> 命名的 headless 服务。当 persistent volume claim 删除时，动态端点和服务是自动删除的。
>

#### PVC举例

- storage-class的名字,需要与上面storageclass的名字一致
- accessModes---k8s不会真正检查存储的访问模式或根据访问模式做访问限制，只是对真实存储的描述，最终的控制权在真实的存储端。目前支持三种访问模式：
  * ReadWriteOnce – PV以读写权限，并且只能被单个node挂载
  * ReadOnlyMany – PV以只读权限，可以被多个node挂载
  * ReadWriteMany – PV以读写权限，可以被多个node挂载
- **Reclaim---当前支持的回收策略:**
  * Retain – 保留数据，需要手工处理
  * Recycle – 回收空间，简单清除文件的操作，删除PV上的数据 (“rm -rf /thevolume/*”)
  * **Delete – 删除PV**
- STATUS
  + Available – PV可以被使用
  + Bound – PV被绑定到PVC
  + Released – 被绑定的PVC被删除，可以被Reclaim
  + Failed – 自动回收失败
- **storage: 最小1Gi**

```yaml
cat <<EOF | tee pvc-sample.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
  namespace: glusterfs
  annotations:
    volume.beta.kubernetes.io/storage-class: "glusterfs"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

查看创建的pvc和pv

```bash
$ kubectl get pvc|grep myclaim
NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim                     Bound     pvc-e98e9117-3ed7-11e8-b61d-08002795cb26   1Gi        RWO            slow           28s

$ kubectl get pv|grep myclaim
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                               STORAGECLASS   REASON    AGE
pvc-e98e9117-3ed7-11e8-b61d-08002795cb26   1Gi        RWO            Delete           Bound     default/myclaim                     slow                     1m
```

## 设置为默认Glusterfs

这样平台分配存储的时候可以自动从glusterfs集群分配pv

```bash
$ kubectl patch storageclass glusterfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
---------------------------------------------------
storageclass.storage.k8s.io "glusterfs" patched

# kubectl get sc
NAME             PROVISIONER               AGE
glusterfs (default)   kubernetes.io/glusterfs   18m
```

### POD挂载

```yaml
        volumeMounts:
          - name: config-vol
            mountPath: /etc/gitlab
            subPath: "config"    
      volumes:
        - name: config-vol
          persistentVolumeClaim:
            claimName: gitlab-config-pvc
```



# 容量限额测试

查看mysql的pod

```bash
$ kubectl get pod|grep mysql
mysql-6599fd55dd-fslkr   1/1     Running   0          91s
```

进入mysql所在容器

```bash
$ kubectl exec -it mysql-6599fd55dd-fslkr /bin/bash
```

查看挂载路径，查看挂载信息

```bash
root@mysql2-mysql-56d64f5b77-j2v84:/# cd /var/lib/mysql
root@mysql2-mysql-56d64f5b77-j2v84:/var/lib/mysql# df -h
Filesystem                                           Size  Used Avail Use% Mounted on
overlay                                               92G  5.6G   87G   7% /
tmpfs                                                 64M     0   64M   0% /dev
tmpfs                                                7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root                               92G  5.6G   87G   7% /etc/hosts
shm                                                   64M     0   64M   0% /dev/shm
192.168.10.113:vol_1cce15bac34f1cb2a8c1c043ee773921 1014M  254M  761M  25% /var/lib/mysql
tmpfs                                                7.9G   12K  7.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                7.9G     0  7.9G   0% /proc/acpi
tmpfs                                                7.9G     0  7.9G   0% /proc/scsi
tmpfs                                                7.9G     0  7.9G   0% /sys/firmware
```

测试mysql

```bash
mysql -pjrtz0755
show databases;
use mysql;
create table my_id(id int);
insert into my_id(id) values(111);
select * from my_id;
```

查看volume

```bash
kubectl exec -it glusterfs-2dpkq gluster volume status
#查看限额
kubectl exec -it glusterfs-2dpkq gluster volume quota vol_7b379dbd1da13ec47d16885e4a16061a list
kubectl exec -it glusterfs-2dpkq gluster volume quota vol_7b379dbd1da13ec47d16885e4a16061a enable
kubectl exec -it glusterfs-2dpkq gluster volume quota vol_7b379dbd1da13ec47d16885e4a16061a limit-usage / 1GB 76%
```

删除PVC，同时删除volume, 暂时不用

```bash
kubectl delete pvc myclaim
```



使用dd写入数据，写入一段时间以后，空间满了，会报错（报错信息有bug，不是报空间满了，而是报文件系统只读，应该是glusterfs和docker配合的问题）, 确定硬盘的最佳块大小：

```bash
root@mysql2-mysql-56d64f5b77-j2v84:/var/lib/mysql# dd if=/dev/zero of=test.img bs=8M count=300 

dd: error writing 'test.img': No space left on device
dd: closing output file 'test.img': No space left on device
```

查看写满以后的文件大小

```bash
root@mysql-6599fd55dd-7cn7l:/var/lib/mysql# ls -l
total 977691
-rw-r----- 1 mysql mysql        56 Apr 17 13:42 auto.cnf
-rw------- 1 mysql mysql      1675 Apr 17 13:42 ca-key.pem
-rw-r--r-- 1 mysql mysql      1107 Apr 17 13:42 ca.pem
-rw-r--r-- 1 mysql mysql      1107 Apr 17 13:42 client-cert.pem
-rw------- 1 mysql mysql      1679 Apr 17 13:42 client-key.pem
-rw-r----- 1 mysql mysql       696 Apr 17 13:51 ib_buffer_pool
-rw-r----- 1 mysql mysql  50331648 Apr 17 13:51 ib_logfile0
-rw-r----- 1 mysql mysql  50331648 Apr 17 13:42 ib_logfile1
-rw-r----- 1 mysql mysql  79691776 Apr 17 13:51 ibdata1
-rw-r----- 1 mysql mysql  12582912 Apr 17 13:51 ibtmp1
drwxr-s--- 2 mysql mysql      4096 Apr 17 13:42 mysql
drwxr-s--- 2 mysql mysql      4096 Apr 17 13:42 performance_schema
-rw------- 1 mysql mysql      1679 Apr 17 13:42 private_key.pem
-rw-r--r-- 1 mysql mysql       451 Apr 17 13:42 public_key.pem
-rw-r--r-- 1 mysql mysql      1107 Apr 17 13:42 server-cert.pem
-rw------- 1 mysql mysql      1675 Apr 17 13:42 server-key.pem
drwxr-s--- 2 mysql mysql      4096 Apr 17 13:42 sys
-rw-r--r-- 1 root  mysql 808189952 Apr 17 13:55 test.img
```

查看挂载信息（挂载信息显示bug，应该是glusterfs的bug）,100%

```bash
root@mysql-6599fd55dd-tqphk:/var/lib/mysql# df -h
Filesystem                                           Size  Used Avail Use% Mounted on
overlay                                               92G  5.6G   87G   7% /
tmpfs                                                 64M     0   64M   0% /dev
tmpfs                                                7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root                               92G  5.6G   87G   7% /etc/hosts
shm                                                   64M     0   64M   0% /dev/shm
192.168.10.113:vol_83ba1634f59ee9da1e64a5b5f42f77c4 1014M 1014M     0 100% /var/lib/mysql
tmpfs                                                7.9G   12K  7.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                7.9G     0  7.9G   0% /proc/acpi
tmpfs                                                7.9G     0  7.9G   0% /proc/scsi
tmpfs                                                7.9G     0  7.9G   0% /sys/firmware
```

查看文件夹大小，为1G

```bash
root@mysql-6599fd55dd-tqphk:/var/lib/mysql# du -h
25M	./mysql
821K	./performance_schema
492K	./sys
969M	.
```

如上说明glusterfs的限额作用是起效的，限制在1G的空间大小。

#### 扩容测试

```bash
$ kubectl get pvc,pv
NAMESPACE   NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
glusterfs   persistentvolumeclaim/myclaim   Bound    pvc-67a377c1-65a4-11e9-b5db-000c29be3271   1Gi        RWO            glusterfs      6m2s

NAMESPACE   NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
            persistentvolume/pvc-67a377c1-65a4-11e9-b5db-000c29be3271   1Gi        RWO            Delete           Bound    glusterfs/myclaim   glusterfs               5m46s
# 修改PVC容量为2Gi
$ kubectl apply -f pvc-sample.yml
# 再次查看，扩容成功，变为2Gi
$ kubectl get pvc,pv
# 进入mysql，再进行测试，可否插入数据，成功

```



---


## 常见问题

### 重启heketi报错解决

#### 报错如下：

```bash
[heketi] ERROR 2018/12/14 11:57:51 heketi/apps/glusterfs/app.go:185:glusterfs.NewApp: Heketi was terminated while performing one or more operations. Server may refuse to start as long as pending operations are present in the db.
```

####  解决

- 修改heketi.json，添加 `"brick_min_size_gb" : 1,`

#### 删除secret并重建

```bash
[root@k8s-master01 kubernetes]# kubectl delete secret heketi-config-secret
[root@k8s-master01 kubernetes]# kubectl create secret generic heketi-config-secret --from-file heketi.json
```

#### 更改heketi的deployment
```bash
 # env添加变量如下
- name: HEKETI_IGNORE_STALE_OPERATIONS
  value: "true"
```

## GFS容器无法启动

#### 报错如下：

```bash
glusterd.service - GlusterFS, a clustered file-system server Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled; vendor preset: disabled
```

- 解决(新建集群，无数据)：

```bash
rm -rf /var/lib/heketi/
rm -rf /var/lib/glusterd
rm -rf /etc/glusterfs/
yum remove glusterfs -y
yum install glusterfs glusterfs-fuse -y
```

参考：

- https://www.cnblogs.com/dukuan/p/9954094.html
- https://www.kubernetes.org.cn/3893.html