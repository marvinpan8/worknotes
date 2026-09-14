# helm 部署 apache superset

官网：https://superset.apache.org/admin-docs/installation/kubernetes

```properties
helm repo add superset https://apache.github.io/superset
helm repo update
helm search repo superset

cd /k0s/superset
helm pull superset/superset --version 0.22.4
tar -zxvf superset-0.22.4.tgz
```

# values.yaml

```properties
global:
  security:
    allowInsecureImages: true
extraSecretEnv:
  SUPERSET_SECRET_KEY: itggZ+IbfmEe7Q3obWx5mhl3IxOWpef30OGkAmx6jHgZev3qjkGo1V0V
bootstrapScript: |
  #!/bin/bash
  uv pip install .[postgres] .[mysql] &&\
  if [ ! -f ~/bootstrap ]; then echo "Running Superset with uid {{ .Values.runAsUser }}" > ~/bootstrap; fi
image:
  repository: 10.10.10.102:5000/apache/superset
  tag: 6.1.0
postgresql:
  image:
    registry: 10.10.10.102:5000
    repository: bitnamilegacy/postgresql
    tag: "14.17.0-debian-12-r3"
redis:
  image:
    registry: 10.10.10.102:5000
    repository: bitnamilegacy/redis
    tag: 7.0.10-debian-11-r4
```

# helm 部署

- kubectl create ns superset

```properties
cd /k0s/superset
helm -n superset install superset ./superset -f values.yaml --wait
# 删除
# helm -n superset delete superset
```

# 额外安装

```properties
apt update 
apt install -y gcc python3-dev pkg-config default-libmysqlclient-dev libpq-dev
uv pip install .[mysql]
```

