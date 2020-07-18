# GlusterFS + Heketi 入门（非容器）

崔秀龙---最近更新于 2019 年 12 月 1 日  3 分钟阅读时长 

GlusterFS 是个开源的分布式文件系统，而 Heketi 在其上提供了 REST 形式的 API，二者协同为 Kubernetes 提供了存储卷的自动供给能力。

一般对这个系统的介绍，都是基于 Docker 的容器内完成的，个人爱好原因，还不太习惯把这个事情放到集群里面，所以介绍一下用 Yum 方式的安装过程。

我们使用三台服务器作为存储集群，操作系统为 CentOS 7。另外假设每台 Gluster FS 服务器挂在有名为 /dev/sdc 的裸设备，安装过程需要有互联网连接。

- Heketi 服务器：10.211.55.31
- Gluster 服务器：
  - 10.211.55.31
  - 10.211.55.32
  - 10.211.55.33

## Heketi 的安装和初始设置

这个很简单，CentOS 的 EPEL Repository 中就提供了他的安装包。

```
yum install -y heketi heketi-client
```

安装之后，会生成 Heketi 的 Service，建立 /etc/heketi，并在其中生成一个叫 heketi.json 的配置文件。这里提供一个样本：

```yaml
{
  "port": "7070",
  "use_auth": false,
  "jwt": {
    "admin": {
      "key": "My Secret"
    },
    "user": {
      "key": "My Secret"
    }
  },
  "glusterfs": {
    "executor": "ssh",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },
    "executor": "ssh",
    "db": "/var/lib/heketi/heketi.db",
    "loglevel": "debug"
  }
}
```

这个简单的配置文件说明：

- 在 7070 提供服务。
- 数据库保存在 `/var/lib/heketi/heketi.db`
- 关闭认证
- 利用 ssh 和 GlusterFS 集群成员进行通信
- ssh 证书保存在 /etc/heketi/heketi_key

既然提到证书，就用 ssh-keygen 来生成一套：

**用户名是注释，您可以删除它或使用-C选项**

```
ssh-keygen -t rsa -q -f /etc/heketi/heketi_key -N '' -C noname
```

后面将会使用这套证书来完成对 GlusterFS 的控制。

> 注意，这里要保证上面提到的**数据库、配置以及证书文件**，一定要确认 Heketi 用户有权进行访问。

## GlusterFS 安装

- 启用仓库：`yum install -y centos-release-gluster`
- 安装软件：`yum install -y glusterfs-server`
- 启用服务：`systemctl enable glusterfs-server`
- 启动服务：`systemctl start glusterfs-server`

注意这里要把上个步骤生成的公钥（heketi_key.pub）加入到本机的信任列表中，例如

```
cat heketi_key.pub >> /root/.ssh/authorized_key
```

## 集群初始化

Heketi 对存储的拓扑结构是这样的：

```
- Topology
    - Cluster a
        - Node a1
            - Device a11
            - Device a12
        - Node a2
    - Cluster b
```

所以初始化过程就按照从上到下的方式来进行：

### 建立集群

```
heketi-cli create cluster
```

创建成功后，会显示一个集群 ID。

### 加入 Node

```
heketi-cli node add --cluster=[clusterid] \
--management-host-name=[node-host] \
--storage-host-name=[node-host] \
--zone=1
```

运行成功会显示新加入的 Node 的 Node ID。

> Add Node 过程失败可能需要查看一下防火墙

### 加入 Device

```
heketi device add \
--name=/dev/sdc
--host=[host-id]
```

### 自动一点

下面的脚本会把运行参数中指定的第一参数作为主机地址，第二参数作为设备名称加入第一个集群

```bash
#!/bin/sh
export HEKETI_CLI_SERVER=http://127.0.0.1:7070
CLUSTER_ID=`heketi-cli cluster list | tail -n 1 | xargs `
CLUSTER="--cluster=$CLUSTER_ID"
HOST="--management-host-name=$1 --storage-host-name=$1"
ZONE="--zone=1"
NODE_ID=`heketi-cli node add $CLUSTER $HOST $ZONE | grep -v -i "Cluster" | grep -i "id" | cut -d : -f 2 | xargs`
heketi-cli device add --name=$2 --node=$NODE_ID
```

> 命令需要用 ‘-s’ 开关指定操作的 Heketi 服务地址。 可以用环境变量来简化一下： export HEKETI_CLI_SERVER=“[http://127.0.0.1:7070"。](http://127.0.0.1:7070"%E3%80%82/)

## Topology

利用 `heketi-cli topology info`，会输出当前的集群结构。而且也可以用 JSON 格式导入和导出整个 Topology。下面的例子供参考：

```json
{
    "volumes": [],
    "nodes": [{
      "zone": 1,
      "hostnames": {
        "manage": ["10.211.55.19"],
        "storage": ["10.211.55.19"]
      },
      "cluster": "f6e6de7dc99ca3ed627e2ab3ae68f9ac",
      "id": "95d3d4fec82be4d2a55ae0aa17344af5",
      "state": "online",
      "devices": [{
        "name": "/dev/sdc",
        "storage": {
          "total": 33419264,
          "free": 33419264,
          "used": 0
        },
        "id": "e4e1b97d38ed5ae70323458c1b8e57b5",
        "state": "online",
        "bricks": []
      }]
    }, {
      "zone": 1,
      "hostnames": {
        "manage": ["10.211.55.21"],
        "storage": ["10.211.55.21"]
      },
      "cluster": "f6e6de7dc99ca3ed627e2ab3ae68f9ac",
      "id": "ab36d04dbface40904a05c33f3fd9800",
      "state": "online",
      "devices": [{
        "name": "/dev/sdc",
        "storage": {
          "total": 33419264,
          "free": 33419264,
          "used": 0
        },
        "id": "a33dee6fd8355c6aa9ff5e2783ecef49",
        "state": "online",
        "bricks": []
      }]
    }, {
      "zone": 1,
      "hostnames": {
        "manage": ["10.211.55.20"],
        "storage": ["10.211.55.20"]
      },
      "cluster": "f6e6de7dc99ca3ed627e2ab3ae68f9ac",
      "id": "bfd478cb0a0a562386c06967fb2b31bc",
      "state": "online",
      "devices": [{
        "name": "/dev/sdc",
        "storage": {
          "total": 33419264,
          "free": 33419264,
          "used": 0
        },
        "id": "24c5a97ccad5b3fc35977bc7419c27ee",
        "state": "online",
        "bricks": []
      }]
    }],
    "id": "f6e6de7dc99ca3ed627e2ab3ae68f9ac"
  }]
}
```