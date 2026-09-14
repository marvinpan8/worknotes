# OceanBase 集群部署





## 部署模式

本文采用三副本部署模式，推荐使用四台机器，您可以根据自己实际情况选择合适的部署方案。本文中四台机器的使用情况如下：

| 角色              | 机器                               | 备注                                                         |
| ----------------- | ---------------------------------- | ------------------------------------------------------------ |
| **obd**           | 10.10.10.4                         | 安装在中控机上的自动化部署软件                               |
| **OBServer 节点** | 10.10.10.1                         | OceanBase 数据库 Zone1                                       |
| **OBServer 节点** | 10.10.10.2                         | OceanBase 数据库 Zone2                                       |
| **OBServer 节点** | 10.10.10.3                         | OceanBase 数据库 Zone3                                       |
| **ODP**           | 10.10.10.1、10.10.10.2、10.10.10.3 | OceanBase 数据库专用的反向代理软件                           |
| OBAgent           | 10.10.10.1、10.10.10.2、10.10.10.3 | OceanBase 数据库监控采集框架                                 |
| obconfigserver    | 10.10.10.4                         | 可提供 OceanBase 的元数据注册，存储和查询服务                |
| Prometheus        | 10.10.10.4                         | 一个开源的服务监控系统和时序数据库，其提供了通用的数据模型以及快捷数据采集、存储和查询接口 |
| Grafana           | 10.10.10.4                         | 一款开源的数据可视化工具，它可以将数据源中的各种指标数据进行可视化展示，以便更直观地了解系统运行状态和性能指标 |

### 内核参数调整

```properties
# 修改内核异步 I/O 限制
fs.aio-max-nr = 1048576

# 网络优化
net.core.somaxconn = 2048
net.core.netdev_max_backlog = 10000 
net.core.rmem_default = 16777216 
net.core.wmem_default = 16777216 
net.core.rmem_max = 16777216 
net.core.wmem_max = 16777216

net.ipv4.ip_forward = 0 
net.ipv4.conf.default.rp_filter = 1 
net.ipv4.conf.default.accept_source_route = 0 
net.ipv4.tcp_syncookies = 1 
net.ipv4.tcp_rmem = 4096 87380 16777216 
net.ipv4.tcp_wmem = 4096 65536 16777216 
net.ipv4.tcp_max_syn_backlog = 16384 
net.ipv4.tcp_fin_timeout = 15 
net.ipv4.tcp_slow_start_after_idle = 0

vm.swappiness = 0
vm.min_free_kbytes = 2097152
vm.overcommit_memory = 0

fs.file-max = 6573688
fs.pipe-user-pages-soft = 0

# 修改进程可以拥有的虚拟内存区域数量
vm.max_map_count = 655360

# 设置 core 文件的文件名格式以及目录
kernel.core_pattern = /data/core-%e-%p-%t
```

