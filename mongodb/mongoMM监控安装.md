# mongoMM监控安装

wget https://downloads.mongodb.com/on-prem-mms/rpm/mongodb-mms-7.0.16.500.20250703T0919Z.x86_64.rpm

## RPM安装

```bash
yum install mongodb-mms-7.0.16.500.20250703T0919Z.x86_64.rpm -y

vim /usr/lib/systemd/system/mongodb-mms.service
```

## 启动

systemctl daemon-reload  && systemctl enable mongodb-mms&& systemctl start mongodb-mms

systemctl status mongodb-mms


## 查看集群和日志

```
emqx ctl cluster status
journalctl -u emqx -f -n 500
systemctl status emqx
```

---

## 登陆界面

- 浏览器输入：`http://172.17.0.30:18083`
- 默认账号 / 密码：`admin/public`