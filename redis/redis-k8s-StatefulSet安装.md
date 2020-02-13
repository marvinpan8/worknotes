#### 节点主机修改系统参数

  否则启动 redis 报错

- `vim /sys/kernel/mm/transparent_hugepage/enabled`

- `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- `vim /etc/rc.local` 增加上面的执行语句
- `chmod +x /etc/rc.d/rc.local`
- `reboot`
- 重新检查`vim /sys/kernel/mm/transparent_hugepage/enabled`

#### 创建configmap

```bash
k create cm redis-cm --from-file=./redis.conf -n comtest
```

#### 创建PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-test-pvc
  namespace: comtest
  annotations:
    volume.beta.kubernetes.io/storage-class: "glusterfs"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 创建StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis
  name: redis
  namespace: comtest
spec:
  ports:
  - name: web
    port: 6379
  clusterIP: None
  selector:
    app: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: redis
  name: redis
  namespace: comtest
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: redis
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  serviceName: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      nodeSelector:
        node-role.kubernetes.io/comtest: comtest
      tolerations:
      - effect: NoSchedule
        key: node
        operator: Equal
        value: comtest
      containers:
      - image: harbor-test.szidc-k8s01.investoday.net/base/redis:5.0.4-alpine
        name: redis
        args: ["/usr/local/etc/redis/redis.conf"]
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 1
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 6379
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 1
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 6379
          timeoutSeconds: 1
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: data-vol
          mountPath: /data
          subPath: redis
        - name: conf-vol
          mountPath: /usr/local/etc/redis
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data-vol
        persistentVolumeClaim:
          claimName: redis-test-pvc
      - name: conf-vol
        configMap:
          name: redis-cm
          items:
          - key: redis.conf
            path: redis.conf
```

