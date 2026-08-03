# helm部署FluentBit 日志收集器（废弃）

- **废弃原因：**因 **LGTM**（Loki, Grafana, Tempo, Mimir）技术栈统一使用 S3对象存储更有优势而废弃。

```properties
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update fluent
helm search repo fluent-bit

kubectl create ns fluent
cd /k0s/fluent-bit
helm pull fluent/fluent-bit-collector --version 1.0.9
tar -zxvf fluent-bit-collector-1.0.9.tgz

# 执行本地安装
helm -n fluent install fluent-bit-collector ./fluent-bit-collector-1.0.9 -f values.yaml --wait
# 删除
# helm -n fluent delete fluent-bit-collector
```

## 使用临时调试容器

```properties
kubectl debug -it fluent-bit-collector-kkrhx --image=10.10.10.102:5000/yauritux/busybox-curl:latest --target=collector
------------------------------------------------------------------------------------
curl -v 10.10.10.102:9000
------------------------------------------------------------------------------------
ll /data/fluent-bit/data
sudo rm -rf /data/fluent-bit/data
sudo mkdir /data/fluent-bit/data
ll /data/fluent-bit
```

## values.yaml 配置

```properties
env:
  - name: AWS_ACCESS_KEY_ID
    value: "3Z2KMaRzTfknwU6ncfL8"
  - name: AWS_SECRET_ACCESS_KEY
    value: "95KhSimGiCHAv7MDeHNwpkAlxk2QsW4lwkj2ERNz"
  - name: AWS_EC2_METADATA_DISABLED
    value: "true"
storage:
  # -- If `true`, writeable host filesystem storage will be enabled.
  enabled: true
  hostPath: /data/fluent-bit/data
config:
  service:
    log_level: info
    http_listen: 0.0.0.0
    # 以下为新增的存储配置
    storage.path: /fluent-bit/data
    storage.sync: normal          # 同步写入模式：normal 或 full
    storage.checksum: off         # 是否启用校验和
    storage.max_chunks_up: 128    # 内存中最大块数量
    storage.backlog.mem_limit: 5M # 回退到内存的 backlog 数据大小限制
  pipeline:
    inputs:
      - name: tail
        alias: k8s-logs
        path: /var/log/containers/*.log
        exclude_path: /var/log/containers/*_kube-system_*.log,/var/log/containers/*_default_*.log,/var/log/containers/*_fluent_*.log
    outputs:
      - name: s3
        match: "*"
        bucket: gin-prod-logs
        region: us-east-1
        endpoint: http://10.10.10.102:9000
        store_dir: /fluent-bit/data/s3-cache   # 本地缓冲目录
        total_file_size: 10M  # 多大触发上传 
        upload_timeout: 5m   # 多久触发上传 
```

