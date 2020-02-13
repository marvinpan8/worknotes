```bash
[xieshuang@VM_177_101_centos gitlab]$ vim /etc/gitlab/gitlab.rb

#13行的 http >> https
external_url 'https://gitlab.szidc-k8s01.investoday.net'

#修改nginx配置 810行
nginx['redirect_http_to_https'] =true
nginx['ssl_certificate'] = "/etc/gitlab/ssl/server.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/server.key"
```

```bash
#进入节点142，执行
docker cp /k8s/gitlab/ssl  cd8ac6ad86db:/etc/gitlab/
#进入gitlab docker，执行
gitlab-ctl reconfigure
```

