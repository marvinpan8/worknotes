# HSM-Lunaclient-docker部署

- ubantu:  24.04
- LunaClient:  10.9.1

## 安装 mini-Lunaclient

https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/install/client_install/linux_docker_minimal_extended.htm

- 在 192.168.1.24 机器上
- 610-000401-015_SW_Linux_Luna_Minimal_Client_V10.9.1_RevA.tar

## 构建 Luna-client docker


- Dockerfile

```properties
FROM 192.168.1.118:80/openjdk:11.0.16

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG container_user=ekemp
ARG container_user_group=ekemp
ARG container_user_uid=1001
ARG container_user_gid=1001

# install hsm luna
ARG MIN_CLIENT
COPY $MIN_CLIENT.tar /tmp
RUN mkdir -p /usr/local/luna/config \
    && tar xvf /tmp/$MIN_CLIENT.tar --strip 1 -C /usr/local/luna \
    && cp /usr/local/luna/jsp/64/libLunaAPI.so /usr/local/openjdk-11/lib/ \
    && chmod 664 /usr/local/openjdk-11/lib/libLunaAPI.so
    
COPY cert /usr/local/luna/cert
COPY Chrystoki.conf /usr/local/luna/config

ENV ChrystokiConfigurationPath=/usr/local/luna/config
ENV PATH="/usr/local/luna/bin/64:${PATH}"

# install packages and create user
RUN apt-get -y update \
    && apt-get install -y sudo\
    && groupadd -g ${container_user_gid} ${container_user_group} \
    && useradd -u ${container_user_uid} -g ${container_user_group} -s /bin/sh -m ${container_user} \
    && adduser ${container_user} sudo
```
- 192.168.1.118:80/ekemp/luna-pci-client:10.9.1

```properties
docker build -t 192.168.1.118:80/ekemp/luna-pci-client:10.9.1 --build-arg MIN_CLIENT=610-000401-015_SW_Linux_Luna_Minimal_Client_V10.9.1_RevA .
```


## 构建 keymanager docker

```properties
FROM 192.168.1.118:80/ekemp/luna-client:10.9.1

ENV active_profile_env=ekemp
ENV loader_path_env=/usr/local/luna/jsp

WORKDIR /home/ekemp

COPY kernel-keymanager-service-1.2.0.1.jar .

# change permissions of file inside working dir
RUN chown -R ekemp:ekemp /home/ekemp

# select container user for all tasks
USER 1001:1001

EXPOSE 8088

ENTRYPOINT ["java", "-Dloader.path=${loader_path_env}", "-Dfile.encoding=UTF-8", "-Dspring.profiles.active=${active_profile_env}",  "-jar", "kernel-keymanager-service-1.2.0.1.jar"]
```




## 测试验证
```properties
docker run --device=/dev/k7pf0  -it --rm --name test --entrypoint "/bin/bash" 10.10.10.102:5000/ekemp/luna-pci-client:10.9.1

# Docker 容器内 java 测试脚本
java -cp ".:/usr/local/luna/jsp/LunaProvider.jar" LunaPkcs11AttributesDemo

# 拷贝 luna 命令
cp /usr/local/luna/bin/64/* /usr/local/bin/
# 查看安装信息
sudo vtl supportInfo
vim c_supportInfo.txt
# 登录查看Available HSMs
sudo lunacm
# 报错 lunacm: error while loading shared libraries: libcap.so.2
apt install -y libcap2
# 登录输入秘钥 ykenXSW@co
role login -n co
# 查看key
par con
exit
```

## 部署执行keygen

```properties
docker run -it --rm --name test -u ekemp --device=/dev/k7pf0 --entrypoint "/bin/bash" 10.10.10.102:5000/ekemp/keys-generator:1.2.0.1

java -Dloader.path=/usr/local/luna/jsp -Dfile.encoding=UTF-8 -Dspring.profiles.active=prod -jar keys-generator-1.2.0.1.jar
```

## k8s 代理验证swagger

```properties
# 到 102 节点 执行
kubectl -n gin-prod port-forward --address 10.10.10.102 keymanager-59455bd87b-5fwcg 8088:8088
# 到 104 节点 执行
kubectl -n gin-prod port-forward --address 10.10.10.104 keymanager-59455bd87b-5fwcg 8088:8088
```

## 删除key

```properties
# 单单进入容器
docker run -it --rm --name test -u ekemp --device=/dev/k7pf0 --entrypoint "/bin/bash" 10.10.10.102:5000/ekemp/keys-generator:1.2.0.1
# 拷贝 RemoveKeyReal.class
docker cp RemoveKeyReal.class test:/home/ekemp
# 执行删除，args 是 label(id),可以有多个
java -cp .:/usr/local/luna/jsp/LunaProvider.jar RemoveKeyReal 122e2054-9402-40d4-8037-049a83e33c77
```

## 更新用户组 hsmusers GID

```properties
# 检查 GID 1002 是否可用
getent group 1002
# 修改 GID：
sudo groupmod -g 1002 hsmusers
# 更新文件所有权（如果有文件属于该组）
sudo find /dev/k7pf0 -gid 986 -exec chgrp 1002 {} \;
# 验证修改：
getent group hsmusers
id ekemp
```

















# ~~集成k8s Device Plugin（跳过）~~

### 1. 为 102 104 节点添加标签

```properties
kubectl label node app102 hsm-node=true
kubectl label node app104 hsm-node=true
```

### 2. 创建 DaemonSet

```properties
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: thales-hsm-plugin
  namespace: kube-system
  labels:
    app.kubernetes.io/name: thales-hsm-plugin
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: thales-hsm-plugin
  template:
    metadata:
      labels:
        app.kubernetes.io/name: thales-hsm-plugin
    spec:
      nodeSelector:
        # hsm-node: "true"
        kubernetes.io/hostname: app102
      priorityClassName: system-node-critical
      hostPID: true
      tolerations:
      - operator: "Exists"
        effect: "NoExecute"
      - operator: "Exists"
        effect: "NoSchedule"
      containers:
      - image: 10.10.10.102:5000/squat/generic-device-plugin
        args:
        - --device
        - '{"name": "thales-hsm", "groups": [{"paths": [{"path": "/dev/k7pf0"}]}]}'
        name: thales-hsm-plugin
        resources:
          requests:
            cpu: 50m
            memory: 10Mi
          limits:
            cpu: 50m
            memory: 20Mi
        ports:
        - containerPort: 8080
          name: http
        securityContext:
          privileged: true
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
        - name: dev
          mountPath: /dev
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
      - name: dev
        hostPath:
          path: /dev
          type: Directory
  updateStrategy:
    type: RollingUpdate
```



### 3. 查看

```properties
kubectl describe nodes | grep -A10 "Allocatable"
```



### 4. 删除 device plugin

```properties
# 1. 删除现有的 DaemonSet
kubectl delete daemonset thales-hsm-plugin -n kube-system
# 2. 删除设备插件注册文件（在每个节点上执行）注意：这需要节点访问权限
ls -l /var/lib/kubelet/device-plugins/
rm -f /var/lib/kubelet/device-plugins/'gdp-c3F1YXQuYWkvdGhhbGVzLWhzbQ==-1774866232.sock'
sudo rm -f /var/lib/kubelet/device-plugins/kubelet_internal_checkpoint
# 3. 重启 kubelet（在每个节点上执行），等待10秒
systemctl restart kubelet
# 4. 检查节点资源（应该看到资源消失或减少）
kubectl describe nodes | grep -A10 "Allocatable"
kubectl describe nodes | grep -A10 "Capacity"
```

