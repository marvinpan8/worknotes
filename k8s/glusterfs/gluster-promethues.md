## 获取最新版二进制文件
在谷歌云上在线生成如下Docker镜像获取二进制文件

```dockerfile
FROM golang:stretch as build

# Otherwise make runs with /bin/sh
ENV SHELL "bash"

RUN \

# Install dep
    curl -sL https://github.com/golang/dep/releases/download/v0.5.0/dep-linux-amd64 > $GOPATH/bin/dep && \
    chmod 755 $GOPATH/bin/dep && \
# Install gometalinter
    curl -sL https://github.com/alecthomas/gometalinter/releases/download/v2.0.11/gometalinter-2.0.11-linux-amd64.tar.gz | tar --strip-components=1 -zxf - -C $GOPATH/bin && \
# Install gluster-prometheus
    mkdir -p $GOPATH/src/github.com/gluster && \
    cd $GOPATH/src/github.com/gluster && \
    git clone https://github.com/gluster/gluster-prometheus.git && \
    cd gluster-prometheus && \
    bash -c "make" && \

    cp ./extras/conf/gluster-exporter.toml.sample /tmp/gluster-exporter.toml && \
    cp ./build/gluster-exporter /tmp/

FROM alpine

COPY --from=build /tmp/gluster-exporter /
COPY --from=build /tmp/gluster-exporter.toml /
```

重新生成 `gluster-centos`镜像，部署即可，暴露端口8080

```dockerfile
FROM gluster/gluster-centos:gluster4u1_centos7
MAINTAINER pantj <pantj@investoday.com.cn>

COPY src/ /gluster-exporter/
COPY gluster-exporter.service /usr/lib/systemd/system/
COPY authorized_keys /root/.ssh/
RUN  chmod 600 /root/.ssh/authorized_keys \
        && chmod a+x /gluster-exporter/gluster-exporter \
        && systemctl enable gluster-exporter.service
```

