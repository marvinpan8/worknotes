# helm 部署 Alloy-k8s-monitoring

###  创建命名空间
```properties
kubectl create ns monitoring
```

## 下载 helm chart
```properties
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo |grep -E 'grafana/k8s-monitoring'
# ------下载最新版本-----------------------------------------------------
cd /k0s/grafana
helm pull grafana/k8s-monitoring --version 4.3.2
tar -zxvf k8s-monitoring-4.3.2.tgz
```

## values.yaml
```properties
cluster:
  name: dc1-prod
```

## helm 部署

- GitHub: https://github.com/grafana/k8s-monitoring-helm
- grafana官网介绍：https://grafana.com/docs/loki/latest/send-data/k8s-monitoring-helm/
- Alloy配置：https://grafana.com/docs/alloy/latest/reference/components/loki/loki.process

```properties
cd /k0s/grafana
helm -n monitoring install k8s-monitoring ./k8s-monitoring -f values-k8s-monitoring.yaml --wait
# 删除
# helm -n monitoring delete k8s-monitoring
# 删除所有pvc
# k -n monitoring delete pvc --all
# 更新升级
helm upgrade -n monitoring k8s-monitoring ./k8s-monitoring -f values-k8s-monitoring.yaml
```

### 修改 _integration_loki_metrics.tpl

- **所在目录 charts/feature-integrations/templates/ _integration_loki_metrics.tpl** 

```properties
// the loki-mixin expects the job label to be namespace/component
      rule {
        source_labels = ["__meta_kubernetes_namespace","__meta_kubernetes_pod_label_app_kubernetes_io_component"]
        separator = "/"
        # 添加以下2行，加 loki- 前缀
        regex = "(.+)/(.+)"
        replacement = "${1}/loki-${2}"
        target_label = "job"
      }
--------------------------------------------------------------------------------------
    prometheus.relabel "loki" {
      forward_to = argument.forward_to.value
      max_cache_size = coalesce(argument.max_cache_size.value, 100000)

      # 固定添加 app_instance=loki 标签
      rule {
        action = "replace"
        replacement = "loki"
        target_label = "app_instance"
      }
```

### 增加监控指标

- 所在目录 charts\feature-integrations\default-allow-lists
- **loki.yaml 新增所有 loki_canary_ 开头指标**
- **mimir.yaml**

```properties
process_network_transmit_bytes_total
process_network_receive_bytes_total
```

- **cadvisor.yaml**
- 指标详情：https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md?plain=1#prometheus

```properties
clusterMetrics:  
  cadvisor:
    metricsTuning:
      keepPhysicalNetworkDevices: ["bond0", "eth[0-9].*"]
      keepPhysicalFilesystemDevices: ["sd.+", "mapper/.*"]
      dropEmptyContainerLabels: false
      dropEmptyImageLabels: false
```

