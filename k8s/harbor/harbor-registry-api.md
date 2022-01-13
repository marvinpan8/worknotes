# registry 镜像仓库API访问

## 获取token

```bash
https://harbor-sit.jrtzcloud.cn/service/token?account=admin&service=harbor-registry&scope=repository:invest/fund10:pull,push
```

## 访问

使用 postman 访问，在header 上加上token,

Authorization: Bearer XXXtokenXXX

```bash
https://harbor-sit.jrtzcloud.cn/v2/invest/fund10/manifests/sha256:deccb5e3c693d0a71b3eb24afcbfa6dd442d0df4a3198e94da7588a3853e7cc6
```

