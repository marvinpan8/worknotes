# jar部署

service

```properties
cat << EOF | tee /etc/systemd/system/powerjob.service
[Unit]
Description=PowerJob Service
After=syslog.target network.target

[Service]
PIDFile=/k0s/powerjob/powerjob.pid
User=root
Group=root
WorkingDirectory=/k0s/powerjob
ExecStart=/k8s/jdk1.8.0_461/bin/java -Xmx2048m -Xms2048m -Doms.mongodb.enable=false -Dspring.datasource.core.jdbc-url=jdbc:mysql://127.0.0.1:3306/powerjob-daily?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai -Dspring.datasource.core.username=cx_user -Dspring.datasource.core.password=cx_admin -Doms.storage.dfs.mysql_series.url=jdbc:mysql://127.0.0.1:3306/powerjob-daily?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai -Doms.storage.dfs.mysql_series.username=cx_user -Doms.storage.dfs.mysql_series.password=cx_admin -jar /k0s/powerjob/powerjob-server.jar
ExecStop=/usr/bin/pkill -f powerjob-server.jar
PrivateTmp=true
TimeoutStartSec=0
KillMode=none

[Install]
WantedBy=multi-user.target
EOF
```

#### 启动服务：

```properties
systemctl daemon-reload && systemctl enable powerjob && systemctl start powerjob
```

#### 检查 kube-nginx 服务运行状态

```properties
systemctl status powerjob
# 10010,10086端口会绑定 VIP ，是个BUG
netstat -anp |grep java
journalctl -f -u powerjob
systemctl stop powerjob
```