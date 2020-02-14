# jenkins-ding-pusher
## IP
 14.113的 对外IP是 202.104.150.28，用于创建机器人


## dingding.json

在机器上的2 个docker中增加文件
```bash
cat << EOF | tee /var/jenkins_home/dingding.json
{
    "msgtype": "markdown",
    "markdown": {
        "title": "私有云服务",
        "text": "### 私有云服务\n- 部署状态：成功\n"
    },
    "at": {
        "isAtAll": true
    }
}
EOF
```
## jenkins配置
- token
- 文件地址：/var/jenkins_home/dingding.json

