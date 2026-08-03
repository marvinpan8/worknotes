# helm部署Alloy-Loki-Grafana

- **LGTM（Alloy, Loki, Grafana, Tempo, Mimir）**技术栈统一使用 S3对象存储
- 手动下载helm chart

```properties
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm repo list
helm search repo |grep -E 'grafana/grafana|grafana/alloy|grafana/loki'
# ------下载最新版本-----------------------------------------------------
cd /k0s/grafana
helm pull grafana/alloy  --version 1.11.0
helm pull grafana/loki  --version 7.1.0
helm pull grafana/grafana  --version 10.5.15
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

```properties
cd /k0s/grafana
helm -n grafana install loki ./loki -f values-loki.yaml --wait
# 删除
# helm -n grafana delete loki
```

### Loki 配置日志保留周期

https://github.com/echo-cool-coding/cool-coding/blob/main/docs/observability/loki/13-log-management/1-log-retention-policy.mdx

#### 场景1：基本时间保留

```properties
# loki-config.yaml
limits_config:
  retention_period: 4320h  # 180天全局保留
```

应用配置后，超过30天的日志将自动删除。

#### 场景2：细粒度保留规则

```properties
# loki-config.yaml
limits_config:
  retention_period: 2160h  # 默认90天
  retention_stream:
    - selector: '{env="production"}'
      priority: 1
      period: 8760h  # 生产环境保留1年
    - selector: '{app="temp-service"}'
      priority: 2
      period: 24h    # 临时服务只保留1天
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

# ■■■ 部署Alloy-v1.18.0

- **日志采集，用DaemonSet 模式部署不能 启用集群模式，各自采集本节点的日志**
- **指标采集，需要另外部署一套启用了cluster 模式的 Alloy**

```properties
cd /k0s/grafana
helm -n grafana install alloy ./alloy -f values-alloy.yaml --wait
# 删除
# helm -n grafana delete alloy
```

## 访问UI

```properties
# 将 alloy service 类型改为 LoadBalancer
http://10.10.10.132:12345/
```

---



# ■■■ 部署grafana-12.3.1

### grafana.ini

- 配置 http://docs.grafana.org/installation/configuration
- 参考： https://github.com/echo-cool-coding

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

# 首次启动 PVC PV 创建后 删除 Deployment grafana 的 initContainers 部分
```

### 验证

```properties
# 获取 admin 密码 v4TSmd29CTxoY8Zaqky8FyVdBTidWlNznWL7dPg4 修改 Ekemp@123
kubectl -n grafana get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 访问

- 内网： http://10.10.10.130/grafana

- kcp:  http://8.218.51.188:50016/grafana

```properties
# 配置日志数据源
http://loki-read.grafana.svc.cluster.local:3100
# Add headers
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