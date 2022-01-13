## 1. 记一次数据挂载盘满的修复
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

## 2. 修复 repair

原因：mongodb不正常关闭造成的mongodb被锁定，这算是一个Mongod 启动的一个常见错误，非法关闭的时候，lock 文件没有remove，第二次启动的时候检查到有lock 文件的时候，就报这个错误了。

### 必须先备份

[官网](https://docs.mongodb.com/v4.0/reference/method/db.repairDatabase/)提示：

Before using [`db.repairDatabase()`](https://docs.mongodb.com/v4.0/reference/method/db.repairDatabase/#db.repairDatabase), make a backup copy of the files in the dbpath directory.

### 方式一(禁止使用)

禁止使用客户端使用 repair database 

在 NoSQLBooster for MongoDB 客户端右键选择 repair database 

### 方式二

#### 1）删除 mongod.lock 文件

```bash
$ rm -f /jrtz/mongo/mongodb/db/mongod.lock
```

#### 2) repair方式启动mongodb

```bash
$ nohup mongod -f /jrtz/mongo/mongodb/mongodb.conf --repair >> ~/mongoRepair.log 2>&1 &
```

#### 3) 再启动一次mongodb

这里一定要再启动一次，不然启动 client 端仍然连不到 server

```bash
# /jrtz/mongo/mongodb/bin/mongod -f /jrtz/mongo/mongodb/mongodb.conf
$ systemctl restart mongodb
```

