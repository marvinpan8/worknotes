# KONG部署手顺

## 创建namespace

执行namespace.yaml

## 创建Secret

执行drone.yml

## 安装PG数据库
### 虚机安装

参考【PG12-Centos7安装】

> 若安装konga选择PG数据库，则需要 `9.6`版本，最新版报错konga not works "error: A hook (`orm`) failed to load"），所以konga 采用mysql数据库

---

### k8s安装（不推荐）

- 创建PVC 
- 执行postgres.yaml，创建RC-PG，创建SVC-PG

## 创建secret

执行此文件 setup_certificate.sh

```bash
#!/bin/bash

set -eufo pipefail

tmpcert="tmpcert"
mkdir $tmpcert
cd $tmpcert

### Create a key+certificate for the control plane
cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: kong-control-plane.kong.svc
spec:
  request: $(openssl req -new -nodes -batch -keyout privkey.pem -subj /CN=kong-control-plane.kong.svc | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

kubectl certificate approve kong-control-plane.kong.svc
kubectl -n kongdev create secret tls kong-control-plane.kong.svc --key=privkey.pem --cert=<(kubectl get csr kong-control-plane.kong.svc -o jsonpath='{.status.certificate}' | base64 --decode)
kubectl delete csr kong-control-plane.kong.svc
rm privkey.pem
cd ..
rm -rf $tmpcert
```

## 创建用户角色权限

执行 sa-role.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kong
subjects:
- kind: ServiceAccount
  namespace: kong
  name: kong
```

## 创建控制平面

- 执行 control-plane-deploy.yaml

- 执行 control-plane-svc.yaml

  第一个启动容器可以用以下docker命令代替

  ```bash
  docker run -it --rm --name kongc -e KONG_PG_HOST="192.168.10.225" -e KONG_PG_PASSWORD="kong" aa3a5eecadaa kong migrations bootstrap --vv
  ```

  

## 创建数据平面

- kubectl create cm prometheus-cm --from-file=prometheus-server.conf -n xxx
- 执行 data-plane-deploy.yaml
- 执行 data-plane-service.yaml

## 创建Konga控制台

- **如果IDC有一套kona，多个环境就可以共用一套环境**
- konga-ui.yml 中的NODE_ENV先为`development`，这样才能创建对应的数据库
- 如果是生产环境，等前一步完成后，修改环境变量 NODE_ENV=`production`，
- 执行 konga-ui.yml
- 内网访问或者配置ingress访问
- tip: 密码采用的是 java 的 BCryptPasswordEncoder加密器

## 创建service,route,consumer并绑定hmac
## 用户绑定服务
```bash
curl http://172.22.254.158:8001/consumer-services
curl http://172.22.254.158:8001/consumers
curl http://172.22.254.158:8001/services
curl http://172.22.254.158:8001/consumer-services -H 'Content-Type: application/json' -X POST -d '{"created_at":1579659182,"consumer":{"id":"58c310c9-9fcc-4d81-874b-a477b71d18e0"},"service":{"id":"f3f389fb-f6d0-44a4-bba0-bc5afb5e3d4e"},"name":"jrtzretail$sdp-rest","expired_at":1737511982}'
```

---

## kong.conf配置文件说明

### 数据缓存部分（DATASTORE CACHE）

- **db_update_frequency：7200**

  - **此值确定Kong节点轮询数据库以查找无效事件的频率。较低的值意味着轮询作业将更频繁地执行，但是您的Kong节点将跟上您所应用的更改。较高的值将意味着Kong节点运行轮询作业的时间将减少，并将重点放在代理流量上。**

    **注意:  意味着对Kong的配置更改在集群中传播的时间最长为db_update_frequency秒。**

  - 用数据存储检查更新实体的频率（以秒为单位）。当节点通过AdminAPI创建、更新或删除实体时，其他节点需要等待下一个轮询（由此值配置）来最终清除旧的缓存实体并开始使用新的实体。Frequency (in seconds) at which to check for updated entities with the datastore. When a node creates, updates, or deletes an entity via the Admin API, other nodes need to wait for the next poll (configured by this value) to eventually purge the old cached entity and start using the new one.

- **db_update_propagation：0**

  - **如果数据库本身最终是一致的(即:Cassandra)，则必须配置此值。这是为了确保更改有时间在数据库节点之间传播。设置好后，从轮询作业接收无效事件的Kong节点将延迟缓存的清除，以获得db_update_propagation秒。**

    **如果连接到最终一致的数据库的Kong节点没有延迟事件处理，那么它可以清除其缓存，只缓存未更新的值(因为更改还没有在数据库中传播)!**

    **您应该将此值设置为数据库集群传播更改所需时间的估计值。**

    **注意:设置此值时，对Kong的配置更改在集群中传播的时间最长为db_update_frequency + db_update_propagation秒。**

  - 在一个由多个Kong节点组成的集群中，连接到同一数据库的其他节点不会立即收到节点A删除该服务的通知。虽然该服务已不在数据库中(已被节点A删除)，但它仍然在节点B的内存中。所有节点都执行一个周期性的后台作业，以与其他节点可能触发的更改同步。此作业的频率可通过以下方式配置:
  - 将数据存储中的实体传播到另一个数据中心的复制节点所需的时间（以秒为单位）。当在分布式环境中，如多数据中心Cassandra集群时，此值应该是Cassandra将一行传播到其他数据中心所需的最大秒数。当设置时，此属性将增加Kong传播实体更改所需的时间。单数据中心设置或Postgre SQL服务器不应遭受此类延迟，并且此值可以安全地设置为0。Time (in seconds) taken for an entity in the datastore to be propagated to replica nodes of another datacenter.  When in a distributed environment such as a multi-datacenter Cassandra cluster, this value should be the maximum number of seconds taken by Cassandra to propagate a row to other datacenters. When set, this property will increase the time taken by Kong to propagate the change of an entity. Single-datacenter setups or PostgreSQL servers should suffer no such delays, and this value can be safely set to 0.

- **db_cache_ttl：172800**

  - **Kong缓存数据库实体(命中和未命中)的时间(以秒为单位)。这个Time-To-Live值在Kong节点错过失效事件时起保护作用，以避免它在过期数据上运行太长时间。当到达TTL时，将从其缓存中清除该值，并再次缓存下一个数据库结果。**

    **默认情况下，没有数据基于这个TTL无效(默认值是0)，这通常很好: Kong节点依赖于失效事件，这些事件在数据库存储级别(Cassandra/PosgreSQL)进行处理。如果您担心Kong节点可能因为任何原因错过无效事件，您应该设置TTL。否则，节点可能会在缓存中使用过期值运行一段未定义的时间，直到手动清除缓存或重新启动节点。**

  - 当由此节点缓存时，来自数据存储的实体的寿命（以秒为单位）。数据库丢失（没有实体）也根据此设置缓存。如果设置为0（默认），则此类缓存的实体或丢失将永远不会过期。Time-to-live (in seconds) of an entity from the datastore when cached by this node. Database misses (no entity) are also cached according to this setting. If set to 0 (default), such cached entities or misses never expire.

- **db_resurrect_ttl：30**

  - 时间（以秒为单位），当无法刷新时，数据存储中的陈旧实体应该被复活（例如，数据存储是无法到达的）。当此TTL过期时，将进行新的尝试来刷新陈旧的实体。Time (in seconds) for which stale entities from the datastore should be resurrected for when they cannot be refreshed (e.g., the datastore is unreachable). When this TTL expires, a new attempt to refresh the stale entities will be made.

- **db_cache_warmup_entities: **plugins,services,consumers,consumer_services,hmacauth_credentials

  - 实体将从数据存储中预先加载到Kong启动时的内存缓存中。这加快了使用给定实体的端点的第一次访问。当“services”实体被配置为热身时，其“host”属性中的值的DNS条目也会异步地预先解析。在`mem_cache_size`中设置的缓存大小应该设置为一个足够大的值，以容纳指定实体的所有实例。如果尺寸不够，kong会记录一个警告。Entities to be pre-loaded from the datastore into the in-memory cache at Kong start-up. This speeds up the first access of endpoints that use the given entities. When the `services` entity is configured for warmup, the DNS entries for values in its `host` attribute are pre-resolved asynchronously as well. Cache size set in `mem_cache_size` should be set to a value large enough to hold all instances of the specified entities. If the size is insufficient, Kong will log a warning.

- **mem_cache_size: **默认128m
  - 数据库实体的内存缓存大小。接受的单位是“k”和“m”，最低建议值为几MB。
  - Size of the in-memory cache for database entities. The accepted units are `k` and `m`, with a minimum recommended value of a few MBs.

- **pg_max_concurrent_queries: 默认0**
  - 设置可以在任何给定时间执行的并发查询的最大数量。这一限制是每个工作进程执行的；该节点的并发查询总数将是：`pg_max_concurrent_query*nginx_worker_processes`。默认值为0将消除此并发限制。
  - Sets the maximum number of concurrent queries that can be executing at any given time. This limit is enforced per worker process; the total number of concurrent queries for this node will be will be: `pg_max_concurrent_queries * nginx_worker_processes`. The default value of 0 removes this concurrency limitation.

- **pg_semaphore_timeout：60000**
  - 定义超时（以ms为单位），在Postgre SQL查询信号量资源获取尝试将失败之后的定义。此类故障通常会导致相关的代理或AdminAPI请求失败，并导致HTTP500状态代码失败。关于这种行为的详细讨论可在在线文档中获得。
  - Defines the timeout (in ms) after which PostgreSQL query semaphore resource acquisition attempts will fail. Such failures will generally result in the associated proxy or Admin API request failing with an HTTP 500 status code. Detailed discussion of this behavior is available in the online documentation.