# harbor部署

## 创建PVC
```bash
# 测试环境
kubectl apply -f pvc-harbor-sit.yaml
# 查看卷
kubectl exec -it glusterfs-2dpkq gluster volume status -n glusterfs
```

## 安装harbor
```bash
# SIT环境
# 配置 secret
cd /jrtz/harbor/harbor-sit/ssl
kubectl apply -f secret-ing-tls.yaml
# 修改values.yaml中的 secretName，notarySecretName值为 harbor-sit-ingress-tls
vim /jrtz/harbor/harbor-sit/values.yaml
------------------------------------------------------
# 测试环境
cd /jrtz/harbor/harbor-sit
helm install --namespace harbor-sit --name harbor-sit .
------------------------------------------------------
```
### 下载登录证书,  存储为ca.crt

```bash
kubectl get secrets/harbor-sit-harbor-ingress -n harbor-sit -o jsonpath="{.data.ca\.crt}" | base64 --decode
```

#### 拷贝ssl证书
```bash
# 测试环境
for n in `seq -w 01 06`;do ssh node-$n "mkdir -p /etc/docker/certs.d/harbor-sit.jrtzcloud.cn";done
#将下载下来的harbor CA证书拷贝到每个node节点的 /etc/docker/certs.d/harbor-sit.jrtzcloud.cn目录下
for n in `seq -w 01 06`;do scp ca.crt node-$n:/etc/docker/certs.d/harbor-sit.jrtzcloud.cn/;done
```
### 校验

```bash
# 进入 harbor-registry容器中
$ kubectl exec -it harbor-sit-harbor-registry-8f76d56fb-z2wrm bash
root [ / ]# df -h
192.168.10.113:vol_fc798971b1e11fff8206961fcc79cd63 500G  5.1G  495G   2% /storage
# 进入 glusterfs 容器中
kubectl exec -it glusterfs-2dpkq bash
[root@vmsrv-010-123 /]# df -h
/dev/mapper/vg_449f1d9ca22f455bd6e353a94125fc4c-brick_039870cc1d0a4660dca2c1e873e80b4d  500G  423M  500G   1% /var/lib/heketi/mounts/vg_449f1d9ca22f455bd6e353a94125fc4c/brick_039870cc1d0a4660dca2c1e873e80b4d
# 进入挂载目录 brick下显示5个文件夹
cd /var/lib/heketi/mounts/vg_449f1d9ca22f455bd6e353a94125fc4c/brick_039870cc1d0a4660dca2c1e873e80b4d/brick
--------------------
drwxrwsr-x  2   10000 10000    6 Apr 23 10:06 chartmuseum
drwx------ 19 gluster input 8192 Apr 23 10:06 database
drwxrwsr-x  2   10000 10000   78 Apr 23 13:35 jobservice
drwxrwsr-x  2 gluster 40000   22 Apr 24 06:40 redis
drwxrwsr-x  3   10000 10000   20 Apr 23 11:49 registry
```

### 扩容试验，注意缩容是不允许的

- **glusterfs可以在不停止pod的情况下扩容**

```bash
# 修改pvc容量为600Gi
vim pvc-harbor-sit.yaml
kubectl apply -f pvc-harbor-sit.yaml
# 查看pvc已经修改为600Gi
kubectl get pvc,pv --all-namespaces
# 再次进入 harbor-registry容器中
$ kubectl exec -it harbor-sit-harbor-registry-57cc584dd8-4zr92 bash
# 挂载已经调整为 600G
root [ / ]# df -h
192.168.10.113:vol_fc798971b1e11fff8206961fcc79cd63 600G  6.1G  594G   2% /storage
# 查看卷
kubectl exec -it glusterfs-2dpkq gluster volume status -n glusterfs
```

---

### 共享存储目录5个

harbor-registry

```bash
192.168.10.113:vol_6ea20ff6008f08c7d618536865f1f7af 500G  5.3G  494G   2% /storage
```

harbor-database

```bash
192.168.10.113:vol_6ea20ff6008f08c7d618536865f1f7af  500G  5.4G  495G   2% /var/lib/postgresql/data
```

harbor-chartmuseum

```bash
192.168.10.113:vol_6ea20ff6008f08c7d618536865f1f7af 500G  5.3G  494G   2% /chart_storage
```

harbor-redis

```bash
192.168.10.113:vol_6ea20ff6008f08c7d618536865f1f7af 500G  5.3G  494G   2% /var/lib/redis
```

harbor-jobservice

```bash
192.168.10.113:vol_6ea20ff6008f08c7d618536865f1f7af 500G  5.3G  494G   2% /var/log/jobs
```


## harbor强制删除

```bash
helm delete harbor-sit --purge
```

---

## k8s使用harbor

#### 指令创建secret

当从私有仓库harbor中pull镜像的时候，k8s集群使用类型为docker-registry的Secret进行认证。
现在创建一个Secret，名称为regcred：

```bash
kubectl create secret docker-registry  chenjb 
 --namespace=<NAME_SPACE>
 --docker-server=<your-registry-server> 
 --docker-username=<your-name> 
 --docker-password=<your-pword> 
 --docker-email=<your-email>
 -------------------------------------------------
 kubectl create secret docker-registry drone --namespace=drone --docker-server=harbor-sit.jrtzcloud.cn --docker-username=drone --docker-password=Invest0755)&%%  --docker-email=drone@investoday.com.cn
```

k8s  yaml文件中增加

```bash
imagePullSecrets:
- name: drone
```

