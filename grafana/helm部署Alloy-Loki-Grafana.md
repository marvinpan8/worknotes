# helm部署Alloy-Loki-Grafana

- **LGTM（Alloy, Loki, Grafana, Tempo, Mimir）**技术栈统一使用 S3对象存储
- 中文官网：https://grafana.org.cn/docs
- 手动下载helm chart

```properties
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
helm search repo |grep -E 'grafana/grafana|grafana/alloy|grafana/loki'
# ------下载最新版本-----------------------------------------------------
cd /k0s/grafana
helm pull grafana/alloy  --version 1.11.0
# 版本旧 helm pull grafana/loki  --version 7.1.0
helm pull grafana-community/loki --version 18.7.3
# 版本旧 helm pull grafana/grafana  --version 10.5.15
helm pull grafana-community/grafana  --version 12.10.3
tar -zxvf alloy-1.11.0.tgz && tar -zxvf loki-7.1.0.tgz && tar -zxvf grafana-10.5.15.tgz 
```

## 创建ns

- 【102机器】

```properties
kubectl create ns grafana
```
---


# ■■■ 部署Loki-3.6.8

- 官网：https://grafana.com/docs/loki/latest/configuration/#common_config
- 部署模式：**SimpleScalable 模式(每天最大 1TB/day )**，包含 read, write, backend，
- **只有write 组件有WAL机制，不能删除PVC**
- **搜索Grafana Dashboard:  helm chart的 src/dashboards 目录下**

```properties
cd /k0s/grafana
helm -n grafana install loki ./loki -f values-loki.yaml --wait
# 删除
# helm -n grafana delete loki
# 删除所有pvc
# k -n grafana delete pvc --all
# 更新升级
helm upgrade -n grafana loki ./loki -f values-loki.yaml
```

### Loki 配置日志保留周期

参考：https://github.com/echo-cool-coding/cool-coding/blob/main/docs/observability/loki/13-log-management/1-log-retention-policy.mdx

#### 保留规则

```properties
# loki-config.yaml
limits_config:
  retention_period: 4380h  # 默认兜底180天
  retention_stream:
  # 保留期越短，数值越小，优先级越高
    - selector: '{namespace=~"default|kubernetes-dashboard"}'
      priority: 1
      period: 24h    # 只保留1天
    - selector: '{namespace="gin-prod"}'
      priority: 10
      period: 8760h  # 生产环境保留1年
```
> enableStatefulSetAutoDeletePVC= true
> 当 PV 已经处于 Released 状态，并且你想让它被同一个名称的 PVC 再次绑定时，就必须删除旧的 PV 对象（磁盘数据会保留），然后重新创建一个全新的 PV（名称可以相同也可以不同）。因为一个处于 Released 状态的 PV 无法被任何 PVC 重新绑定。

### 校验

```properties
# 缓存 DNS 检查
nslookup -type=SRV _memcached-client._tcp.loki-chunks-cache.monitoring.svc.cluster.local
nslookup -type=SRV _memcached-client._tcp.loki-results-cache.monitoring.svc.cluster.local
```

---

Loki 在写入日志时，有**两层检查**：

| 检查层级             | 配置项                                         | 检查逻辑                                         |
| -------------------- | ---------------------------------------------- | ------------------------------------------------ |
| **第一层：全局检查** | `reject_old_samples_max_age: 168h`             | 如果日志时间戳 < (当前时间 - 168h)，**拒绝**     |
| **第二层：流级检查** | `unordered_writes: true` + `max_chunk_age: 2h` | 如果日志时间戳 < (该流最新时间戳 - 1h)，**拒绝** |

### 监控指标

- 在 src/dashboard 目录下新增 loki-canary-dashboard.json

### 修改 PrometheusRule 的时间窗口

- **templates/monitoring/rules/loki_rules.yaml  全部替换[1m] 为 [2m]，否则没有数据**





# ■■■ ~~部署Alloy-ds-v1.18.0(跳过)~~

- 待 k8s-monitoring 集中部署，所以跳过
- 中文官网：https://grafana.org.cn/docs/alloy/latest/set-up/deploy/#use-kubernetes-statefulsets
- **日志采集，用DaemonSet 模式部署 不能用集群模式，各自采集本节点的日志。**
- **k8s日志配置：https://grafana.org.cn/docs/alloy/latest/collect/logs-in-kubernetes/**
- **指标采集，需要另外部署一套启用了cluster 模式的 Alloy**
- **配置官网**：https://grafana.com/docs/alloy/latest/reference/components
- **配置官网中文**：https://grafana.org.cn/docs/alloy/latest/reference/components

```properties
cd /k0s/grafana
helm -n grafana install alloy-ds ./alloy -f values-alloy-ds.yaml --wait
# 删除
# helm -n grafana delete alloy-ds
# 更新升级
helm upgrade -n grafana alloy-ds ./alloy -f values-alloy-ds.yaml
```
- **自动创建了CRD：PodLogs** 

---



# ■■■ 部署grafana-13.1.2

### grafana.ini

- 配置 http://docs.grafana.org/installation/configuration
- 参考： https://github.com/echo-cool-coding
- 搜索 dashboard: https://grafana.com/grafana/dashboards/

```properties
grafana.ini:
  server:
    root_url: http://10.10.10.131/grafana
    serve_from_sub_path: true
  plugins:
    preinstall_auto_update: false
  analytics:
    reporting_enabled: false
    check_for_updates: false
    check_for_plugin_updates: false
```

```properties
cd /k0s/grafana
helm -n grafana install grafana ./grafana -f values-grafana.yaml --wait
# 删除
# helm -n grafana delete grafana
# 更新升级
helm upgrade -n grafana grafana ./grafana -f values-grafana.yaml
```

- **首次启动 若带 PVC 创建后 必须删除 Deployment grafana 的 initContainers 部分，才能重启deploy**

### 验证

```properties
# 获取 admin 密码 v4TSmd29CTxoY8Zaqky8FyVdBTidWlNznWL7dPg4 修改 Ekemp@123
kubectl -n grafana get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 访问

- 内网： http://10.10.10.130/grafana

- kcp:  http://8.218.51.188:50016/grafana

```properties
# 配置数据源
loki-gin-prod     http://loki-gateway.grafana.svc.cluster.local
loki-k8s-prod     http://loki-gateway.grafana.svc.cluster.local
prometheus        http://mimir-gateway.mimir.svc:80/prometheus
# Add headers
X-Scope-OrgID = dc1-gin-prod
X-Scope-OrgID = dc1-k8s-prod
# 打开 Label browser 选择 namespace pod
```

#### 检索

在「Label Filters」下方的「Line contains」模块中，「Text to find」输入框就是用来检索日志内容里的字段 / 关键词的。比如你想找日志里包含 “error” 的内容，直接在这个输入框 里填 “error” 即可。

如果需要更复杂的字段检索（比如精确匹配、正则、提取字段），可以点击界面右上角的「Code」标签（当前是「Builder」模式），直接写**LogQL 查询语句**。

```properties
{pod="c1"} |= "你要找的字段"  # 包含某个字段
{pod="c1"} |~ "正则匹配的字段"  # 正则匹配字段
{pod="c1"} | json | .status  # 解析JSON日志并提取status字段
```



# ■■■  jsonnet 生成 dashboard json


### 安装 jb  jsonnet

```properties
sudo apt install golang-go
go version
go install -a github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb@latest
go install github.com/google/go-jsonnet/cmd/jsonnet@latest
```

###  先下载包，再转换 libsonnet  为 json

```properties
cd /k0s/grafana/loki-mixin
jb install
# 生成 vendor 文件夹
ll vendor
# 修改 配置文件
vim config.libsonnet
# ---新增---------------
canary: {
  enabled: true
},
# 创建 入口文件
cat > canary-entry.libsonnet << 'EOF'
local config = import 'config.libsonnet';
local canary = import 'dashboards/loki-canary-dashboard.libsonnet';

config + canary + {
  _config+:: {
    canary+: {
      enabled: true,
    },
  },
}
EOF
# 执行转换
jsonnet -J vendor -J lib canary-entry.libsonnet > loki-canary-dashboard.json
```

- **将生成的文件放入 helm chart 的 src/dashboards 目录下，用于自动生成**

- **修改 chart的  templates\monitoring\dashboards 目录下，新增记录**

```properties
  "loki-canary-dashboard.json": |
    {{ $.Files.Get "src/dashboards/loki-canary-dashboard.json" | fromJson | toJson }}
```

- **grafana 最终存储到 grafana 数据库 resource表中**

```properties
SELECT * FROM `resource` where `group` = 'dashboard.grafana.app' and `value` like '%Loki%'
```

