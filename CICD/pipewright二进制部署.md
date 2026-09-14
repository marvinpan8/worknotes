# pipewright 二进制部署

GitHub官网：https://github.com/huangchengsir/pipewright

二进制文件：https://github.com/huangchengsir/pipewright/releases/download/v1.5.3/pipewright_1.5.3_linux_amd64.tar.gz

部署文件：https://raw.githubusercontent.com/huangchengsir/pipewright/master/install.sh

- 删除下载包逻辑，改成手动下载
- 因需要使用docker cri，最好使用二进制安装
## 配置

##### 创建数据目录：mkdir -p /mnt/dockerImageData/cicd/pipewright/data/
##### vim /etc/pipewright/pipewright.env

```properties
# Pipewright 运行配置(systemd EnvironmentFile);改后 systemctl restart pipewright 生效。
PIPEWRIGHT_ADDR=192.168.1.118:7080
PIPEWRIGHT_ADMIN_PASSWORD=admin123
PIPEWRIGHT_MASTER_KEY_FILE=/etc/pipewright/master.key
PIPEWRIGHT_DB=/mnt/dockerImageData/cicd/pipewright/data/pipewright.db
```


## 部署

```properties
bash install.sh
cd /mnt/dockerImageData/cicd/pipewright
# 修改配置文件权限
sudo chmod 644 /etc/pipewright/pipewright.env
# 启动停止
systemctl start pipewright
systemctl stop pipewright
# 状态
systemctl status pipewright
```
