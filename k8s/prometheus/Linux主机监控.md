# prometheus 监控linux服务器

**node_exporter：用于\*NIX系统监控，使用Go语言编写的收集器**。

- **使用版本:** node_exporter 1.1.2

- 使用文档：https://prometheus.io/docs/guides/node-exporter/
- GitHub：https://github.com/prometheus/node_exporter
- exporter列表：https://prometheus.io/docs/instrumenting/exporters/

**安装监控客户端**

**1、下载**

```
https://github.com/prometheus/node_exporter/releases/tag/v1.1.2
```

**2、解压压缩包**

```
tar -zxvf node_exporter-1.1.2.linux-amd64.tar.gz 
```

**3、移动并进入目录**

```
mv node_exporter-1.1.2.linux-amd64 /usr/local/bin/node_exporter
cd /usr/local/bin/node_exporter
```

**4、启动node_exporter服务，默认9100端口**

```
./node_exporter
```

常用参数：

```properties
# 收集文件系统，忽略哪些不搜集
--collector.filesystem.ignored-mount-points="^/(dev|proc|sys|var/lib/docker/.+)($|/)"  
# 管理的系统服务
--collector.systemd.unit-whitelist=".+"
# 指定监听端口, 默认9100
--web.listen-address=":9100"
```

**5、添加系统服务：**

vim /etc/systemd/system/node_exporter.service

```properties
[Unit]
Description=Linux prometheus metrics
Documentation=https://github.com/prometheus/node_exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**6、启动添加后的系统服务**

```bash
systemctl daemon-reload && systemctl enable node_exporter && systemctl restart node_exporter
```

**7、查看导出器导出的数据信息**

```bash
curl http://localhost:9100/metrics
```

**8、ServiceMonitor（已存在可忽略）**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: linux-group
  name: linux-group
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: http
    path: /metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: linux-group
```

**9、Service & Endpoints**

在Endpoints上加上IP 即可

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: linux-group
  name: linux-group
  namespace: monitoring
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http
    port: 9100
    targetPort: 9100
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: linux-group
  name: linux-group
  namespace: monitoring
subsets:
- addresses:
  - ip: 192.168.10.230
  - ip: 192.168.10.232
  ports:
  - name: http
    port: 9100
    protocol: TCP
```

