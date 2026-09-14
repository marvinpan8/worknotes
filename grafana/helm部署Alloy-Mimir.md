# helm部署Alloy-Mimir

###  创建命名空间
```properties
kubectl create ns mimir
```

# ■■■ 部署Mimir-3.1.2

## values.yaml

```properties
image:
  repository: 10.10.10.102:5000/grafana/mimir
  tag: 3.1.2
mimir:
  structuredConfig:
    # 禁用kafka
    ingest_storage:
      enabled: false  
    ingester:
    # 禁用kafka, 开启 grpc 推送
      push_grpc_method_enabled: true
```

### 租户配置

```properties
runtimeConfig:
  overrides:
    dc1-k8s-prod:
      max_global_series_per_user: 1500000
      # 监控指标保存 s3 期限 180 天后硬删除
      compactor_blocks_retention_period: 180d    
```

## ■■ helm 部署

- 官网helm(6.1.0)：https://grafana.org.cn/docs/mimir/latest/set-up/helm-chart/ 
- 参数配置：https://grafana.com/docs/mimir/latest/references/configuration-parameters/

```properties
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo |grep -E 'grafana/mimir-distributed'
# ------下载最新版本-----------------------------------------------------
cd /k0s/grafana
helm pull grafana/mimir-distributed --version 6.1.0
tar -zxvf mimir-distributed-6.1.0.tgz

cd /k0s/grafana/mimir-distributed/charts/rollout-operator
vim values.yaml 
# 修改 镜像仓库
image:
  registry: 10.10.10.102:5000
# 部署 mimir
cd /k0s/grafana
helm -n mimir install mimir ./mimir-distributed -f values-mimir.yaml --wait
# 删除
# helm -n mimir delete mimir
# 删除所有pvc
# k -n mimir delete pvc --all
# 更新升级
helm upgrade -n mimir mimir ./mimir-distributed -f values-mimir.yaml
```
## 获取监控指标metrics

```properties
kubectl get svc
cd /k0s/grafana/deploy/mimir/metrics
kubectl -n kubernetes-dashboard exec -it busybox-curl-5f68bb4fdf-xthjb -- curl 10.244.109.222:3500/metrics > ./loki-canary.conf
```

### 修改mimir 固定指标名

```properties
cd /k0s/grafana/mimir-distributed/mixins/dashboards
# 全局替换
container_network_receive_bytes_total ==> process_network_receive_bytes_total
container_network_transmit_bytes_total ==> process_network_transmit_bytes_total
```

### 修改prometheus rule 的时间窗口

- **mixins/rules.yaml  全部替换[1m] 为 [2m]，否则没有数据**

## 调试网络 tcpdump(跳过)

- 调试：tcpdump -i eth0 port 8080 -n -v -A

```properties
securityContext:
  capabilities:
  add:
    - NET_ADMIN
    - NET_RAW
    - SETUID
    - SETGID
  drop:
    - ALL
  readOnlyRootFilesystem: false
  allowPrivilegeEscalation: false
```

# ■■■ 部署Prometheus operator

从 [Releases 页面](https://github.com/prometheus-operator/prometheus-operator/releases/latest) 下载对应版本的 `bundle.yaml`，以确保 CRD 与 Operator 版本匹配。

- kubectl create ns prometheus
- 修改 namespace: default => prometheus
- 替换镜像：docker pull quay.io/prometheus-operator/prometheus-operator:v0.93.0

```properties
kubectl create -f bundle.yaml
# 校验
kubectl get all -n prometheus
kubectl get ServiceMonitor -A
kubectl get PodMonitor -A
```



# ■■■ 部署Alloy-sts-v1.18.0

- **指标采集，用 sts-cluster 模式**
- 中文官网：https://grafana.org.cn/docs/alloy/latest/set-up/deploy/#use-kubernetes-statefulsets
- **配置官网**：https://grafana.com/docs/alloy/latest/reference/components
- **配置官网中文**：https://grafana.org.cn/docs/alloy/latest/reference/components

```properties
cd /k0s/grafana
helm -n mimir install alloy-sts ./alloy -f values-alloy-sts.yaml --wait
# 删除
# helm -n mimir delete alloy-sts
# 更新升级
helm upgrade -n mimir alloy-sts ./alloy -f values-alloy-sts.yaml
```

- **自动创建了CRD：PodLogs** 

## 访问UI

```properties
# 将 alloy service 类型改为 LoadBalancer
http://10.10.10.132:12345/
```

## grafana dashboard

- **在源代码的 operations/alloy-mixin/rendered/dashboards 文件夹下**

























---


## ■■ Mimir 各组件介绍

### 1. 入口和协调

| 组件              | 功能                             | 关键特性                                                     |
| ----------------- | -------------------------------- | ------------------------------------------------------------ |
| **mimir-gateway** | **统一网关**：所有外部流量的入口 | 路由写入请求到Distributor，查询请求到Query Frontend；可配置Nginx/Gateway |

### 2. 写入链路（Ingest Path）

| 组件                          | 功能                                                         | 关键特性                                                     |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **mimir-distributor（2）**    | **写入网关**：接收Prometheus Remote Write请求，验证数据格式，进行租户隔离和限流 | 无状态，可水平扩展；将数据分片（通过哈希环）转发给对应的Ingester |
| **mimir-kafka(可选)**         | **写入缓冲队列**：作为Distributor和Ingester之间的缓冲，防止 Ingester 压力过大 | 可选组件：用于异步写入模式，提升写入吞吐量                   |
| **mimir-ingester-zone-a/b/c** | **数据写入和存储**：接收Distributor发来的数据，写入内存并异步刷盘（块存储） | **有状态**，使用PVC持久化；支持**多可用区部署**（zone-a/b/c实现高可用）；数据副本跨zone分布 |


### 3. 查询链路（Query Path）

| 组件                           | 功能                                                         | 关键特性                                               |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| **mimir-query-frontend**       | **查询前端**：接收查询请求，进行分片、缓存、并行化处理       | 无状态；将**大查询**拆分为多个**小查询**；缓存查询结果 |
| **mimir-query-scheduler（2）** | **查询调度器**：管理查询任务队列，将查询请求分发给Querier执行 | 负载均衡；支持查询优先级和限流                         |
| **mimir-querier**              | **查询执行器**：真正执行PromQL查询，从**Ingester（近期数据）和Store Gateway（历史数据）**读取数据 | 无状态；执行查询时合并来自多个存储的数据               |

### 4. 长期存储和后台任务

| 组件                               | 功能                                                     | 关键特性                                         |
| ---------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| **mimir-store-gateway-zone-a/b/c** | **长期存储网关**：从对象存储（S3）读取**历史数据**块     | **有状态，多可用区部署**；支持数据分片和索引查询 |
| **mimir-compactor**                | **数据压缩器**：对对象存储中的数据块进行合并、压缩和去重 | 后台任务；优化存储空间和查询性能                 |
| **mimir-ruler**                    | **规则评估器**：评估Recording Rules和Alerting Rules      | 根据配置定期执行PromQL，生成预聚合数据或触发告警 |

### 5. 告警和管理

| 组件                         | 功能                                                         | 关键特性                                                     |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **mimir-alertmanager（2）**  | **告警管理器**：接收Ruler触发的告警，进行分组、抑制、静默和路由 | **有状态**，支持高可用（2副本）；与Prometheus Alertmanager兼容 |
| **mimir-overrides-exporter** | **配额导出器**：暴露每个租户的限制配置指标（如最大系列数、样本数） | 用于监控和运维，暴露`/metrics`端点                           |
| **mimir-rollout-operator**   | **滚动更新协调器**：协调**有状态**组件的滚动更新             | 确保升级过程中数据不丢失                                     |

### 6. 缓存层

| 组件                          | 功能                                       | 关键特性                               |
| ----------------------------- | ------------------------------------------ | -------------------------------------- |
| **mimir-chunks-cache（3）**   | **数据块缓存**：缓存从对象存储读取的数据块 | 基于Memcached，加速查询                |
| **mimir-index-cache（3）**    | **索引缓存**：缓存对象存储的索引信息       | 加速标签查询                           |
| **mimir-metadata-cache（3）** | **元数据缓存**：缓存数据块的元数据         | 加速查询规划                           |
| **mimir-results-cache（3）**  | **查询结果缓存**：缓存已执行的查询结果     | 由Query Frontend使用，提升重复查询速度 |


