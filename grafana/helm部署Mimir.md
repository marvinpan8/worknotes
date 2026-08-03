# ■■■ 部署Mimir-3.1.2

## ■■ 各组件介绍
### 1. 写入链路（Ingest Path）

| 组件                          | 功能                                                         | 关键特性                                                     |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **mimir-distributor**         | **写入网关**：接收Prometheus Remote Write请求，验证数据格式，进行租户隔离和限流 | 无状态，可水平扩展；将数据分片（通过哈希环）转发给对应的Ingester |
| **mimir-kafka(可选)**         | **写入缓冲队列**：作为Distributor和Ingester之间的缓冲，防止 Ingester 压力过大 | 可选组件：用于异步写入模式，提升写入吞吐量                   |
| **mimir-ingester-zone-a/b/c** | **数据写入和存储**：接收Distributor发来的数据，写入内存并异步刷盘（块存储） | **有状态**，使用PVC持久化；支持**多可用区部署**（zone-a/b/c实现高可用）；数据副本跨zone分布 |


### 2. 查询链路（Query Path）

| 组件                      | 功能                                                         | 关键特性                                               |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| **mimir-query-frontend**  | **查询前端**：接收查询请求，进行分片、缓存、并行化处理       | 无状态；将**大查询**拆分为多个**小查询**；缓存查询结果 |
| **mimir-query-scheduler** | **查询调度器**：管理查询任务队列，将查询请求分发给Querier执行 | 负载均衡；支持查询优先级和限流                         |
| **mimir-querier**         | **查询执行器**：真正执行PromQL查询，从**Ingester（近期数据）和Store Gateway（历史数据）**读取数据 | 无状态；执行查询时合并来自多个存储的数据               |

### 3. 长期存储和后台任务

| 组件                               | 功能                                                     | 关键特性                                         |
| ---------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| **mimir-store-gateway-zone-a/b/c** | **长期存储网关**：从对象存储（S3）读取**历史数据**块     | **有状态，多可用区部署**；支持数据分片和索引查询 |
| **mimir-compactor**                | **数据压缩器**：对对象存储中的数据块进行合并、压缩和去重 | 后台任务；优化存储空间和查询性能                 |
| **mimir-ruler**                    | **规则评估器**：评估Recording Rules和Alerting Rules      | 根据配置定期执行PromQL，生成预聚合数据或触发告警 |

### 4. 告警和管理

| 组件                         | 功能                                                         | 关键特性                                                     |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **mimir-alertmanager**       | **告警管理器**：接收Ruler触发的告警，进行分组、抑制、静默和路由 | **有状态**，支持高可用（2副本）；与Prometheus Alertmanager兼容 |
| **mimir-overrides-exporter** | **配额导出器**：暴露每个租户的限制配置指标（如最大系列数、样本数） | 用于监控和运维，暴露`/metrics`端点                           |
| **mimir-rollout-operator**   | **滚动更新协调器**：协调**有状态**组件的滚动更新             | 确保升级过程中数据不丢失                                     |

### 5. 缓存层

| 组件                     | 功能                                       | 关键特性                               |
| ------------------------ | ------------------------------------------ | -------------------------------------- |
| **mimir-chunks-cache**   | **数据块缓存**：缓存从对象存储读取的数据块 | 基于Memcached，加速查询                |
| **mimir-index-cache**    | **索引缓存**：缓存对象存储的索引信息       | 加速标签查询                           |
| **mimir-metadata-cache** | **元数据缓存**：缓存数据块的元数据         | 加速查询规划                           |
| **mimir-results-cache**  | **查询结果缓存**：缓存已执行的查询结果     | 由Query Frontend使用，提升重复查询速度 |

### 6. 入口和协调

| 组件              | 功能                             | 关键特性                                                     |
| ----------------- | -------------------------------- | ------------------------------------------------------------ |
| **mimir-gateway** | **统一网关**：所有外部流量的入口 | 路由写入请求到Distributor，查询请求到Query Frontend；可配置Nginx/Gateway |

## values.yaml

```properties
mimir:
  structuredConfig:
    # 禁用kafka
    ingest_storage:
      enabled: false  
    ingester:
    # 禁用kafka, 开启 grpc 推送
      push_grpc_method_enabled: true
```



## ■■ helm 部署

- 官网(6.1.0)：https://grafana.org.cn/docs/mimir/latest/set-up/helm-chart/ 
- 参数配置：https://grafana.com/docs/mimir/latest/references/configuration-parameters/

```properties
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm repo list
helm search repo |grep -E 'grafana/mimir-distributed'
# ------下载最新版本-----------------------------------------------------
cd /k0s/grafana
helm pull grafana/mimir-distributed --version 6.1.0
tar -zxvf mimir-distributed-6.1.0.tgz
# 创建命名空间
kubectl create ns mimir

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
```

