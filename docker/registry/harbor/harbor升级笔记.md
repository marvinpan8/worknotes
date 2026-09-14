# harobr升级笔记

## 一、备份数据

- 获取 pod 所在节点

  ```bash
  $ k get po -owide -n harbor-sit
  -------------------------------
  NAME                                               READY   STATUS    RESTARTS   AGE   IP               NODE             NOMINATED NODE   READINESS GATES
  harbor-sit-harbor-database-0                       1/1     Running   0          22d   172.30.43.250    192.168.10.125   <none>           <none>
  ```

- PG 数据库在125机器上，备份数据

  ```bash
  $ docker ps | grep database
  $ docker cp 08709c3978fe:/var/lib/postgresql ~/postgresql
  ```

- redis 在114机器上， 备份数据

  ```bash
  $ docker ps | grep redis
  $ docker cp 07c6ff2daf4c:/var/lib/redis ~/redis
  ```

- chartmuseum 和 jobservice 目录为空，可不备份

- registry 数据达482G，太大了，暂不备份

## 二、版本关系表
| helm | harbor | postgresql | redis |registry|
|--- | --- | ---| --- |---|
| 1.0.1 | 1.7.5 | 9.6.10 | 4.0.10 | v2.6.2 |
| 1.1.6 | 1.8.6 | 9.6.14 | 4.0.14 | v2.7.1 |
| 1.2.4 | 1.9.4 | 9.6.14 | 4.0.14 | v2.7.1.m |
| 1.3.9 | 1.10.9 | 9.6.23 | 4.0.14 | v2.7.1.m |
| 1.4.6 | 2.0.6 | 9.6.20 | 4.0.14 | v2.7.1.m |
| 1.5.5 | 2.1.5 | 9.6.21 | 4.0.14 | v2.7.1.m |
| 1.6.4 | 2.2.4 | 9.6.23 | 4.0.14 | v2.7.1.m |
| 1.7.4 | 2.3.4 | 13.4 | 6.0.16 | v2.7.1.m |

## 三、升级helm

### 1. 修改value.yaml

### 2. 修改 templates\core\core-cm.yaml

### 3. 更新执行命令

如果更新错误，比如`Error: cannot re-use a name that is still in use`，请检查value.yaml 格式是否正确
```bash
helm upgrade harbor-sit --namespace harbor-sit --force .
```
### ~~4. 安装命令（用不上）~~

```bash
# 安装
helm install --namespace harbor-sit --name harbor-sit .
# 删除
helm delete harbor-sit --purge
```

### Core Error log
#### 1.  LDAP 未配置
```properties
2021-11-24T03:32:10Z [ERROR] [/pkg/config/store/store.go:74]: error when loading data item, key ldap_url, value , error the configure value can not be empty
2021-11-24T03:32:10Z [ERROR] [/pkg/config/store/store.go:74]: error when loading data item, key ldap_base_dn, value , error the configure value can not be empty
```
在数据库的 properties 表中新增缺失的配置

#### 2. 其他配置未设置，但是debug 日志， 先忽略

[DEBUG] [/pkg/config/db/db.go:45]:

```bash
failed to get metadata, key:admiral_url, error:<nil>, skip to load item
failed to get metadata, key:clair_db_sslmode, error:<nil>, skip to load item
failed to get metadata, key:count_per_project, error:<nil>, skip to load item
failed to get metadata, key:clair_db_password, error:<nil>, skip to load item
failed to get metadata, key:clair_url, error:<nil>, skip to load item
failed to get metadata, key:clair_db_username, error:<nil>, skip to load item
failed to get metadata, key:with_clair, error:<nil>, skip to load item
failed to get metadata, key:clair_db, error:<nil>, skip to load item
failed to get metadata, key:cfg_expiration, error:<nil>, skip to load item
failed to get metadata, key:clair_db_host, error:<nil>, skip to load item
failed to get metadata, key:clair_db_port, error:<nil>, skip to load item
failed to get metadata, key:reload_key, error:<nil>, skip to load item
```

