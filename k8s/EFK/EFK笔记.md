## POD挂载采集器filebeat

1. 增加`configMap`

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: efk
data:
  filebeat.yml: |
    filebeat.prospectors:
    - input_type: log
      paths:
        - "/log/*"
    setup.template.name: "permission-test"
    setup.template.pattern: "permission-test-*"
    output.elasticsearch:
      hosts: ["elasticsearch-logging.efk:9200"]
      username: "elastic"
      password: "changeme"
      index: "permission-test-%{[beat.version]}-%{+yyyy.MM.dd}"
```

2. 需要监控的pod  yml增加 `image`
```bash
- image: harbor-test.szidc-k8s01.investoday.net/base/filebeat:6.3.2
        name: filebeat
        args: [
          "-c", "/etc/filebeat/filebeat.yml",
          "-e"
        ]
        volumeMounts:
        - name: app-logs
          mountPath: /log
        - name: filebeat-config
          mountPath: /etc/filebeat/
```
3. 增加 `volumes`
```bash
 volumes:
      - name: app-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: filebeat-config
```

4. 原来镜像增加 `volumeMounts`
```bash
volumeMounts:
- name: app-logs
  mountPath: /usr/logs/permission
```

---

LinkedIn有99%的服务都是java service，有15种以上log，最重要的是access log和application log；

我们做的一件事情是通过java Container logger标准化直接写入Kafka。有的程序直接写到kafka，不上磁盘，有的程序还要同时写到磁盘里面，这是可配置的。

这些是通过LinkedIn standard container一起rollout到所有的service上。开发人员什么都不用管，只要在程序里写logger.info/logger.error，那些信息就会直接进到Kafka。对于程序日志，默认警告以上的级别进入Kafka，可以在线通过jmx控制。对于访问日志，我们有10%采样，可以通过ATS入口动态控制。

###  在192.168.10.110管理机器上

```bash
# 获取当前索引
curl -u elastic:changeme '10.254.216.39:9200/_cat/indices?v' 
# 模糊匹配删除
curl -XDELETE -u elastic:changeme http://10.254.216.39:9200/filebeat-*
  {"acknowledged":true}
# 如果存储不够可以设置定时删除，下面是保留3天的日志, crontab -e	
30 2 * * * /usr/bin/curl -XDELETE -u elastic:changeme http://10.254.216.39:9200/*-$(date -d '-3days' +'%Y.%m.%d') >/dev/null 2>&1
# 查看定时任务
$ crontab -l
# 编辑加入定时任务
$ crontab -e
#  重启 crond 服务
$ systemctl restart crond
```



以下是定时删除脚本（暂时未使用）：

```bash
#!/bin/bash
time=$(date -d '-3days' +'%Y.%m.%d')
curl -XDELETE -u elastic:changeme http://10.254.216.39:9200/*-${time}
```

