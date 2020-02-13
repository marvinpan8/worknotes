# docker registry 镜像删除

原文：https://blog.csdn.net/l6807718/article/details/52886546 

registry:2.5.0版本的镜像，将镜像默认存放在了/var/lib/registry 目录下 
/var/lib/registry/docker/registry/v2/repositories/ 目录下会有几个文件夹，命名是已经上传了的镜像的名称。 
如果需要删除已经上传的镜像，现有两种方法

### 官方推荐版重点内容

1) 更改registry容器内/etc/docker/registry/config.yml文件，后`docker restart dockerid`

```yaml
storage:
  delete:
    enabled: true
```

2) 找出你想要的镜像名称的tag

```bash
$ curl -X GET <protocol>://<registry_host>/v2/<镜像名>/tags/list
# eg:
curl -X GET http://10.84.140.92:5000/v2/invest/asset/tags/list
```

3) 拿到digest_hash参数

```bash
$ curl --header "Accept: application/vnd.docker.distribution.manifest.v2+json" -I -X GET http://<仓库地址>/v2/<镜像名>/manifests/<tag>
# eg:
curl --header "Accept: application/vnd.docker.distribution.manifest.v2+json" -I -X GET http://10.84.140.92:5000/v2/invest/asset/manifests/v1.6
```


4) 复制digest_hash

```yaml
Docker-Content-Digest: <digest_hash>
```

5) 删除清单

```bash
$ curl -I -X DELETE <protocol>://<registry_host>/v2/<repo_name>/manifests/<digest_hash>
# eg:
$ curl -I -X DELETE http://10.84.140.92:5000/v2/invest/manifests/sha256:b3a15ef1a1fffb8066d0f0f6d259dc7f646367c0432e3a90062b6064f874f57c
```


6) 删除文件系统内的镜像文件，注意2.4版本以上的registry才有此功能

```bash
$ docker exec -it <registry_container_id> bin/registry garbage-collect <path_to_registry_config>
# eg
$ docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

### 简易版 
1.打开镜像的存储目录，如有-V操作打开挂载目录也可以，删除镜像文件夹

```bash
$ docker exec <容器名> rm -rf /var/lib/registry/docker/registry/v2/repositories/<镜像名>
```

2.执行垃圾回收操作，注意2.4版本以上的registry才有此功能

```bash
$ docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```


