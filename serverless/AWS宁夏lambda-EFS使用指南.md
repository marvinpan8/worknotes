# AWS宁夏 lambda-EFS使用指南

## 前言
### 1. s3桶目录介绍

在 fc-jrtz 桶下  

- **python-layer：python程序依赖包zip文件压缩包，包含在python目录下**
- **src-xxxxxx：函数输入文件xlsx集合，测试时用到，正式时从unzip解压至此文件夹**
- **dst-xxxxxx：函数输出文件xlsx集合，测试时用到，正式时zip压缩至此文件夹**
- **zip-input：函数输入文件xlsx集合zip压缩包，限制最大500个xlsx文件**
- **zip-output：函数输出文件xlsx集合zip压缩包**
### 2. VPC设置
#### 1） 创建互联网网关并将其附加到 VPC

#### 2） 创建 NAT 网关
- 填写名字，选择公网子网（Public Subnet)
- 选择类型公有
- 分配一个弹性IP
#### 3）创建路由
**注意**: 连接 Amazon VPC 的 Lambda 函数在进行请求时随机选择关联的子网。您的函数使用的所有子网应具有相同的配置，以防止 Lambda 使用配置错误的子网造成的随机错误。

1. 在 Amazon VPC 控制台的[路由表](https://console.aws.amazon.com/vpc/home?#RouteTables:sort=routeTableId)中，为 VPC 创建两个[自定义路由表](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#CustomRouteTables)。**提示：**在创建时，对于**名称标签**，添加一个名称以帮助您识别要将路由表关联到哪个子网。例如，将一个子网命名为 **Public Subnet**，将另一个子网命名为 **Private Lambda**。
2. [将公有子网路由表关联到](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-table-assocation) (**Public Subnet**) 您要使其成为公有子网的子网。在此路由表中[添加新路由](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html#AddRemoveRoutes)。指定以下参数：对于**目的地**，输入 **0.0.0.0/0**。对于**目标**，选择**互联网网关**，然后选择您创建的互联网网关 ID (**igw-123example**)。选择**保存路由**。关联的子网现在是一个公有子网。
3. [将私有路由表关联到](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-table-assocation) (**Private Lambda**) 私有子网。在此路由表中[添加新路由](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html#AddRemoveRoutes)。指定以下参数：
   对于**目的地**，输入 **0.0.0.0/0**。对于**目标**，选择 **NAT 网关**，然后选择您创建的 NAT 网关 ID (**nat-123example**)。如果您使用的是 NAT 实例，则选择**网络接口**。选择**保存路由**。

>  **注意：确保** NAT 网关的路由处于**活动**状态。如果 NAT 网关被删除且您没有更新路由，它们将处于**黑洞**状态。有关更多信息，请参阅[更新路由表](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html#nat-gateway-create-route)。

### 3.  EFS文件系统创建
- 填写名称，选择默认VPC
- 可用性和持久性选择**区域**，在多个可用区之间冗余存储数据

## 一、S3上传依赖包和源数据

### 1.  在116 机器上
```bash
cd /code/fc && mkdir xxxxxx_layer
```

### 2.  安装依赖包，按需添加
```bash
pip install --default-timeout=100 --no-cache-dir --upgrade \
    -i https://mirrors.aliyun.com/pypi/simple/ -t ./package \
    numpy==1.17.0 \
    pandas==0.21.1 \
    hmmlearn==0.2.6
# zip 打包
zip -r xxxxxx_layer.zip .
# 上传至 s3://fc-jrtz/python-layer/
aws s3 cp xxxxxx_layer.zip s3://fc-jrtz/python-layer/
```
### 3. S3解压至EFC文件系统

在函数 S3-EFS 配置里操作

- 选择添加**EFS文件系统**和**接入点**，本地挂载路径填写 **/mnt/jrtz**
- 添加 **VPC**， 子网选择 3个 **私有子网 private-lambda-a、private-lambda-b、private-lambda-c**，安全组选择默认的VPC安全组（入站和出站规则都为All）

- 在测试事件里配置测试事件：Records.s3.object.key: "python-layer/hmm_10d_test_layer.zip"

- 点击TEST 执行
- 在 CloudWatch下的Log groups 选择 /aws/lambda/S3-EFS， 点击最近的日志流， 在日志事件栏选择 View as text 查看日志。重点查看依赖包在总路径 /jrtz/python/ 下

### 4. 压缩源数据文件并上传zip-input

```bash
aws s3 cp xxxxxx_layer.zip s3://fc-jrtz/zip-input/
```

> 注意：先测试10、20、50、100个文件的压缩文件测试，再合并正式计算

## 二、函数代码测试

### 1. 配置触发器

- 事件类型选择 S3
- 选择存储桶 fc-jrtz
- 事件类型选择---所有对象创建事件
- 前缀:src-hmm-10d-test/
- 后缀:.xlsx

### 2. 函数代码调试

- 配置环境变量
  - START_DATE：设置半年起点，eg: 2020-01-01
  - END_DATE：设置半年终点，eg: 2020-06-30
  - PYTHONPATH：EFS挂载的依赖包路径，固定 /mnt/jrtz/python

- 修改代码 
  - dst_dir：输出目标文件到s3目录
  - calculator_.run(StockCode_) 函数文件修改
- 先在网页上手动上传一个xlsx文件到 src-xxxxxx 文件夹进行调试
- 主要对内存进行合理地调试设置正确的值。每次测试日志都有 REPORT，比如
```bash
REPORT RequestId: 277ad80e-bbc4-41ef-b03f-fe84a7a1fc8f	Duration: 165906.71 ms	Billed Duration: 165907 ms	Memory Size: 1024 MB	Max Memory Used: 521 MB	Init Duration: 350.60 ms
```
### 3. 解压批量测试

在函数 unzip 配置里操作，VPC和文件系统同函数 S3-EFS 

- 在测试事件里配置测试事件：Records.s3.object.key: "zip-input/hmm_raw_data_500_0.zip"

- 逐个修改500_X，

- 由于AWS并发是 500 递增，最好间隔1分钟再点击 Test 运行

- 在 CloudWatch下的Log groups 选择 /aws/lambda/hmm_10d_test， 点击最近的日志流， 在日志事件栏选择 View as text 查看日志。重点查看运行时间和内存使用。

- 在211机器的 jrtz_auth_dev 库中的 cloud_fc_log 中有提交记录。cloud_type：默认1: aliyun 2:aws香港 3:aws宁夏 4:aliyun容器

  ```sql
  SELECT * FROM `cloud_fc_log` where created_time > '2022-01-11 00:00:00'
  -- ORDER BY created_time DESC
  ORDER BY duration DESC
  ```

## 三、压缩结果文件上传

在函数 zip 配置里操作，VPC和文件系统同函数 S3-EFS 
- zip_name 输出目标文件名

- src_dir 输入源文件s3目录

- zip_dir 输出目标文件s3目录

- 在 CloudWatch下的Log groups 选择 /aws/lambda/zip， 点击最近的日志流， 在日志事件栏选择 View as text 查看日志。重点查看运行时间和内存使用。

- 查看并发执行完成情况

```bash
# 上传的文件数量
aws s3 ls s3://fc-jrtz/dst-hmm-10d-test/ |wc -l
# 完成的文件数量
aws s3 ls s3://fc-jrtz/src-hmm-10d-test/ |wc -l
```

## 四、下半年继续

修改执行函数的环境变量

- START_DATE：设置半年起点，eg: 2020-01-01
- END_DATE：设置半年终点，eg: 2020-06-30
- 重新执行 解压批量测试 和 查看并发执行完成情况
- 重新执行 压缩结果文件上传 步骤