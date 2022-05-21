# AWS中国 lambda-EFS使用指南

## 前言
### 1. s3桶目录介绍

在 fc-jrtz 桶下  

- **python-layer：python程序依赖包zip文件压缩包，包含在python目录下**
- **src-xxxxxx：函数输入文件xlsx集合，测试时用到，正式时从unzip解压至此文件夹**
- **dst-xxxxxx：函数输出文件xlsx集合，测试时用到，正式时zip压缩至此文件夹**
- **zip-input：函数输入文件xlsx集合zip压缩包，限制最大500个xlsx文件**
- **zip-output：函数输出文件xlsx集合zip压缩包**

前提是 S3创建接入点

填写 VPC-ID，并创建接入点策略
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws-cn:s3:cn-northwest-1:947401374981:accesspoint/fc-jrtz/object/*"
        }
    ]
}
```

### 2. VPC设置
#### 1） 创建互联网网关并将其附加到 VPC

​	有些内部服务需要外网访问，需要创建互联网网关，并创建安全组，允许访问指定端口，比如RDS-mysql的3306

#### 2） 创建 NAT 网关（不开外网不需要）
- 填写名字，选择公网子网（Public Subnet)
- 选择类型公有
- 分配一个弹性IP
#### 3）创建路由（不开外网不需要）
**注意**: 连接 Amazon VPC 的 Lambda 函数在进行请求时随机选择关联的子网。您的函数使用的所有子网应具有相同的配置，以防止 Lambda 使用配置错误的子网造成的随机错误。

1. 在 Amazon VPC 控制台的[路由表](https://console.aws.amazon.com/vpc/home?#RouteTables:sort=routeTableId)中，为 VPC 创建两个[自定义路由表](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#CustomRouteTables)。**提示：**在创建时，对于**名称标签**，添加一个名称以帮助您识别要将路由表关联到哪个子网。例如，将一个子网命名为 **Public Subnet**，将另一个子网命名为 **Private Lambda**。
2. [将公有子网路由表关联到](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-table-assocation) (**Public Subnet**) 您要使其成为公有子网的子网。在此路由表中[添加新路由](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html#AddRemoveRoutes)。指定以下参数：对于**目的地**，输入 **0.0.0.0/0**。对于**目标**，选择**互联网网关**，然后选择您创建的互联网网关 ID (**igw-123example**)。选择**保存路由**。关联的子网现在是一个公有子网。
3. [将私有路由表关联到](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-table-assocation) (**Private Lambda**) 私有子网。在此路由表中[添加新路由](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html#AddRemoveRoutes)。指定以下参数：
   对于**目的地**，输入 **0.0.0.0/0**。对于**目标**，选择 **NAT 网关**，然后选择您创建的 NAT 网关 ID (**nat-123example**)。如果您使用的是 NAT 实例，则选择**网络接口**。选择**保存路由**。

>  **注意：确保** NAT 网关的路由处于**活动**状态。如果 NAT 网关被删除且您没有更新路由，它们将处于**黑洞**状态。有关更多信息，请参阅[更新路由表](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html#nat-gateway-create-route)。

#### 4）创建终端节点，内网访问
创建并关联路由表
- S3---Gateway
- dynamodb---Gateway
- ~~SQS---Interface ，**注意：接口类型将按小时数收费**~~

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
```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "cn-north-1",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "EXAMPLE123456789",
        "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "fc-jrtz-bj",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws-cn:s3:::fc-jrtz-bj"
        },
        "object": {
          "key": "python-layer/hmm_10d_test_layer.zip",
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "sequencer": "0A1B2C3D4E5F678901"
        }
      }
    }
  ]
}
```
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

### 2. 配置异步调用重试次数

可以设置0,1,2次， 前两次之间的间隔1分钟，后两次间隔2分钟

### 3. DynamoDB

- 没有时间类型，用整数代替
- 容量按需
- 走VPC需要创建**终端节点**，并关联路由表，该路由表关联3个子网
- 增加 lambda 所有权限
- 在控制台浏览项目中按条件查询，也可以在控制台使用 PartiQL编辑器查询

### 4. 函数代码调试

- 配置环境变量
  - START_DATE：设置半年起点，eg: 2020-01-01
  - END_DATE：设置半年终点，eg: 2020-06-30
  - PYTHONPATH：EFS挂载的依赖包路径，固定 /mnt/jrtz/python

- 修改代码 
  - dst_dir：输出目标文件到s3目录
  - calculator_.run(StockCode_) 函数文件修改

- 先在网页上手动上传一个xlsx文件到 src-xxxxxx 文件夹进行调试

- 或者测试事件测试

```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "cn-northwest-1",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "EXAMPLE123456789BBBB",
        "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "fc-jrtz",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws-cn:s3:::fc-jrtz"
        },
        "object": {
          "key": "src-hmm-10d-test/1_raw_data.xlsx",
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "sequencer": "0A1B2C3D4E5F678901"
        }
      }
    }
  ]
}
```

- 主要对内存进行合理地调试设置正确的值。每次测试日志都有 REPORT，比如
```bash
REPORT RequestId: 277ad80e-bbc4-41ef-b03f-fe84a7a1fc8f	Duration: 165906.71 ms	Billed Duration: 165907 ms	Memory Size: 1024 MB	Max Memory Used: 521 MB	Init Duration: 350.60 ms
```
### 5. unzip 解压批量测试

> 注意，此函数只能单例运行，不可多例运行

在函数 unzip 配置里操作，VPC和文件系统同函数 S3-EFS 

- 配置测试事件：Records.s3.object.key: "zip-input/hmm_raw_data_500_0.zip"
```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "cn-northwest-1",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "EXAMPLE123456789CCCC",
        "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "fc-jrtz",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws-cn:s3:::fc-jrtz"
        },
        "object": {
          "key": "trigger/hmm_raw_data_500_9.zip",
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "sequencer": "0A1B2C3D4E5F678901"
        }
      }
    }
  ]
}
```

- 逐个修改500_X，

- 由于AWS并发是 500 递增，最好间隔1分钟再点击 Test 运行

- 在 CloudWatch下的Log groups 选择 /aws/lambda/hmm_10d_test， 点击最近的日志流， 在日志事件栏选择 View as text 查看日志。重点查看运行时间和内存使用。

- 在211机器的 jrtz_auth_dev 库中的 cloud_fc_log 中有提交记录。cloud_type：默认1: aliyun 2:aws香港 3:aws宁夏 4:aliyun容器

```sql
SELECT * FROM `cloud_fc_log` where created_time > '2022-02-10 00:00:00'
-- ORDER BY created_time DESC
ORDER BY duration DESC
-- DynamoDB 查询
select * from cloud_fc_log where fc_name = 'hmm_10d_test' order by duration desc
```
- 查看并发执行完成情况

```bash
# 上传的文件数量 和 完成的文件数量
aws s3 ls s3://fc-jrtz/src-hmm-10d-test/ |wc -l && aws s3 ls s3://fc-jrtz/dst-hmm-10d-test/ |wc -l
```

## 三、zip压缩结果文件并上传s3

配置测试事件随便hello

在函数 zip 配置里操作，VPC和文件系统同函数 S3-EFS 

- bucket = "fc-jrtz"

- **zip_name 输出目标文件名修改**
- src_dir 输入源文件s3目录
- dst_dir  生成的结果文件s3目录
- zip_dir 输出目标zip文件s3目录
- 在 CloudWatch下的Log groups 选择 /aws/lambda/zip， 点击最近的日志流， 在日志事件栏选择 `View as text` 查看日志。重点查看运行时间和内存使用。
- 下载该 zip 文件

## 四、下半年继续

修改执行函数的环境变量

- **START_DATE：设置半年起点，eg: 2020-01-01**

- **END_DATE：   设置半年终点，eg: 2020-06-30**

- **删除 src-hmm-10d-test 目录，重新创建**

- **删除 dst-hmm-10d-test 目录，重新创建**

- 重新执行 解压批量测试 和 查看并发执行完成情况

- 重新执行 压缩结果文件上传 步骤


## 五、附录

```bash
# 获取预置并发数
aws lambda  get-function-concurrency --function-name hmm_10d_test
```

## 六、参考
Java SDK-https://github.com/awsdocs/aws-doc-sdk-examples

aws cli lambda: https://docs.aws.amazon.com/zh_cn/cli/latest/reference/lambda/index.html

aws cli sqs: https://docs.aws.amazon.com/zh_cn/cli/latest/reference/sqs/receive-message.html

函数示例：https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/samples-blank.html

## 七、 配置异步调用死信队列（按小时收费）
- 创建AWS-SQS服务，只能选择**标准队列**，输入名称，其他默认：消息保留4天，最大消息大小为256KB

- 在AWS-SQS服务中选择刚才队列，点击-**发送和接收消息**-按钮，在**接收消息栏**设置 **最大消息计数**为最大999。点击**轮询消息**可以查看消息

  ```bash
  # 只返回一条消息？
  aws sqs receive-message --queue-url https://sqs.ap-east-1.amazonaws.com/530894007240/lambda --max-number-of-messages 10
  ```

- ~~走VPC需要创建**终端节点**，并关联路由表，该路由表关联3个子网~~

  > 使用 网关终端节点 不会发生任何额外费用，比如 S3 和 DynamoDB
  > [接口终端节点](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html)和 [Gateway Load Balancer 终端节点](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html)  将按小时数收费。

- 配置异步调用死信队列

- 添加执行角色权限，Lambda 需要以下权限来管理 Amazon SQS 队列中的消息，添加全部权限[AmazonSQSFullAccess](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonSQSFullAccess)  