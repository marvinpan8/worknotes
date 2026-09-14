# Dockery-UI 极简部署

官网：https://github.com/bizjs/Dockery/blob/main/README.md

自己维护内网 Docker Registry 的人大概都经历过这个区间: 

- 一边嫌 `distribution/distribution` 裸跑太原始 —— 没 UI、没账户、htpasswd 所有人共用一把钥匙; 
- 一边嫌 Harbor 过重，官方 `docker-compose.yml` 拉起十几个容器,给一个五人小组用配置成本远大于收益。
- **Dockery 就是为这中间地带做的**：一个镜像,一个端口,跑起来就是完整的私有仓库 —— 能 push/pull,有网页,能按人分权限,不依赖 Postgres / Redis / 任何外部服务。

## 不做什么

v0.1 明确不做的东西：镜像扫描、cosign 验签、跨机复制、HA、多租户。每一项都会把"单容器 + 本地盘"这个边界打破 —— 需要这些请上 Harbor，Dockery 不跟它抢场景。

## docker-compose.yml

- https://github.com/bizjs/Dockery/blob/main/docker-compose.ghcr.yml


```properties
services:
  dockery:
    image: 192.168.1.118:80/bizjs/dockery:0.10.0
    container_name: dockery
    restart: unless-stopped
    ports:
      - "5001:5000"
    environment:
      DOCKERY_ADMIN_USERNAME: admin
      DOCKERY_ADMIN_PASSWORD: "admin123"
      REGISTRY_AUTH_TOKEN_REALM: http://192.168.1.118:5001/token
    volumes:
      - /mnt/dockerImageData/dockery/data:/data
```

## 启动

```properties
docker pull ghcr.io/bizjs/dockery:0.10.0
mkdir -p /mnt/dockerImageData/dockery/data
docker compose up -d
```

## 备份

- **registry 是原生 registry 数据包，可共用**

```properties
/data/
├── registry/          镜像 blob(默认 filesystem driver)
├── db/dockery.db      SQLite(users / repo_permissions / audit_log)
└── config/
    ├── jwt-private.pem  Ed25519 私钥(0600),单一真源
    └── jwt-jwks.json    每次启动由私钥派生
```

**备份 = /data 整包**。

- 丢 `jwt-private.pem` → 签出去的 token 全废
- 丢 `dockery.db` → 用户表重置。

> 用 `REGISTRY_STORAGE_*` 切 S3 / OSS / Azure 只会搬 `registry/`,但`db/` 和 `config/` 仍需要挂 `/data`。

## 架构

```properties
           外部 :5001 (host → :5000 container)
                       │
                   [ nginx ]
    ┌────────────┬────────┬────────────┐
    │            │        │            │
   / 静态     /token   /api/*       /v2/*
    │            │        │            │
 web-ui    dockery-api :3001   distribution :5001
                  ▲                    ▲
                 └── manifest PUT ────┘  其它 /v2/* 直达 registry
                 │                    ▲
                 ├── SQLite           │
                 ├── jwt-private.pem  │
                 └── jwt-jwks.json ───┘  registry 用 JWKS 验签
```

