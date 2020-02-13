# Docker中images中none的镜像是否可以删除

Docker中images中none的镜像是否可以删除呢？

担心删除了会有问题啊，小白用户啊。查查资料还是收获不少哦。简要翻译国外的一篇文章哈，不对请指正。

原文：http://www.projectatomic.io/blog/2015/07/what-are-docker-none-none-images/

1. What are `<none>:<none>` images ?  什么是<none>:<none>镜像呢？
2. What are dangling images ?      什么是“临时”还是摇摆镜像？
3. Why do I see a lot of `<none>:<none>` images when I do `docker images -a` ?  为什么会看到一堆的<none>:<none>镜像呢？
4. What is the difference between `docker images` and `docker images -a` ?       docker images和`docker images -a` 看到的有什么不同呢？



文章主要解答这几个问题啦，结合了docker镜像的原理。

这里呢有好的<none>:<none>镜像和坏的<none>:<none>镜像哦。他们分别怎么来的呢？

### 好的<none>:<none>镜像的产生

例如从镜像仓库里拿一个fedora 镜像。如图虽然`docker images` 只显示`fedora:latest，但是docker images -a 显示了两个镜像fedora:latest 和<none>:<none>. `

原来docker中镜像是有垂直父子关系的，层级关系可以在/var/lib/docker/graph中看到。docker pull fedora执行的时候呢，就会每次下载一个镜像。

可以通过查看`/var/lib/docker/graph` 的json查看父子关系。这些镜像都不会引起存储空间占用的问题。

```bash
root@iZ2zejcwx7sfb1o4vvupxkZ:/var/lib/docker/graph# more ff0e2b608af6b1901d8ad9e9556e9e8ffe91b4c5386039e32bdf087df6157f65/json
{"container_config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":fal
se,"OpenStdin":false,"StdinOnce":false,"Env":null,"Cmd":["/bin/sh -c echo 'export PATH=$ORACLE_HOME/bin:$PATH' \u003e\u003e /etc/bas
h.bashrc"],"Image":"","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":null},"created":"2016-04-20T10:29:03.
276290831Z","layer_id":"sha256:a5d9cef8ef2a0ffd19fea965e22924c2717bdcec82f628344111ae5aeec3ec13","parent_id":"sha256:c74e9fd53a7e49d
4d4cd562a69aa8ccc094ee17aedb7cc26a161af2903af8f68"}
root@iZ2zejcwx7sfb1o4vvupxkZ:/var/lib/docker/graph# 
```

### 坏的<none>:<none>镜像的产生

而`docker build` 或是 `pull` 命令就会产生临时镜像。如果我们用dockerfile创建一个helloworld镜像后，因为版本更新需要重新创建，那么以前那个版本的镜像就会成为临时镜像。这个是需要删除的。删除命令见下。

### 清除坏的<none>:<none>镜像

Docker目前没有自动垃圾收集系统。这绝对是一个很好的功能。目前此命令可用于执行手动垃圾回收。

```bash
docker rmi $(docker images -f "dangling=true" -q)
```

#### 定时任务脚本

clearDockerNone.sh

```bash
#!/bin/bash
docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker stop
docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker rm
docker images|grep none|awk '{print $3 }'|xargs docker rmi
```

