## 记一次数据挂载盘满的修复
- 先另外搞一个大一点的空盘
- 拷贝数据到空盘
- unmount 两块盘，重新挂载目录即可
- 目录设置权限  `chown -R mongodb:mongodb db`

```bash
# 重启
rm -f /jrtz/mongo/mongodb/db/mongod.lock && systemctl stop mongodb && systemctl start mongodb
# 修复数据
rm -f /jrtz/mongo/mongodb/db/mongod.lock && mongod -f /jrtz/mongo/mongodb/mongodb.conf --repair
# 删除为修复完记录文件
rm -f /jrtz/mongo/mongodb/db/_repair_incomplete
# 查看日志
tail -f -n 500  /jrtz/mongo/mongodb/log/mongodb.log
# 查看配置文件
vim /jrtz/mongo/mongodb/mongodb.conf
```

