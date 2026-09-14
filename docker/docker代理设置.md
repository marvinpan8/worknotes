# docker 代理设置


## ■ 非官网代理配置
- **不适用 docker.io 官网会拦截代理**
- **比如外网 grc.io等**

```properties
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://8.218.51.188:50099"
Environment="HTTPS_PROXY=http://8.218.51.188:50099"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.1.118,registry.develop.ekemp.com.cn"
EOF
# 重启
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl show --property=Environment docker
```

## ■ 删除非官网代理

```properties
sudo rm -f /etc/systemd/system/docker.service.d/http-proxy.conf
# 重启
sudo systemctl daemon-reload
sudo systemctl restart docker
```



---
## ■ docker.io 官网代理配置

- **配置国内加速网址：vim /etc/docker/daemon.json**
- **daocloud 官网：** https://github.com/DaoCloud/public-image-mirror

```properties
{
  "registry-mirrors": [
      "https://docker.m.daocloud.io",
      "https://docker.xuanyuan.me",
      "https://docker.tbedu.top"
   ],
  "insecure-registries": ["192.168.1.118:80"],
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}

# 重启
sudo systemctl daemon-reload
sudo systemctl restart docker
```





