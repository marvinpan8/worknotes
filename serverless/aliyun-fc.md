# aliyun-fc使用笔记

## 打包上传

[官网Python运行环境---自定义模块章节](https://help.aliyun.com/document_detail/56316.html?spm=a2c4g.11186623.6.577.36c35d5cwJyezX)

打包时，需要针对**文件**进行打包，而不是针对代码整体目录进行打包。打包完成后，**入口函数文件需要位于包内的根目录。**

- 在Windows下打包时，可以进入函数代码目录，全选所有文件以后，单击鼠标右键，选择**压缩为zip包**，生成代码包。
- 在Linux下打包时，通过调用zip命令时，将源文件指定为代码目录下的所有文件，实现生成部署代码包，例如`zip code.zip /home/code/*`。


## 使用pip包管理器进行依赖管理

通过 `pip install -t .` 命令安装依赖库至函数根目录下。上传代码库时将依赖库一同打包上传。下文以安装PyMySQL 库为例进行详细介绍。

建立一个目录，用于存放 **代码和依赖模块**

```bash
mkdir /tmp/code
# 在/tmp/code目录下安装依赖。
cd /tmp/code
pip install -t . PyMySQL
```

## 内置模块

[官网Python运行环境---内置模块章节](https://help.aliyun.com/document_detail/56316.html?spm=a2c4g.11186623.6.577.36c35d5cwJyezX)

除了Python的标准模块，函数计算的Python运行环境中还包含了一些常用模块，您可以直接引用，目前包含的模块如下所示。

> **注意：前面6个内置的科学计算库版本号，以免冲突，不必自己安装**
>
> **解决报错：ImportError: numpy.core.multiarray failed to import**
>
> **pandas 必须选择 0.21.1 兼容 numpy >= 1.7.0**

| 模块名称                     | 模块介绍                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| numpy 1.13.3                 | [科学计算库](http://www.numpy.org/)                          |
| scipy 1.0.0                  | [科学计算库](https://www.scipy.org/)                         |
| matplotlib 2.0.2             | [绘图库](https://matplotlib.org/)                            |
| scrapy 1.4.0                 | [数据抓取库](https://scrapy.org/)                            |
| wand 0.4.4                   | [图片处理库](http://docs.wand-py.org/en/0.4.4/)              |
| opencv 3.3.0.10              | [计算机视觉库](http://opencv-python-tutroals.readthedocs.io/en/latest/py_tutorials/py_setup/py_intro/py_intro.html) |
| oss2 2.6.0                   | [OSS SDK](https://github.com/aliyun/aliyun-oss-python-sdk)   |
| tablestore 4.6.0             | [表格存储SDK](https://github.com/aliyun/aliyun-tablestore-python-sdk) |
| aliyun-fc2 2.1.0             | [函数计算SDK](https://github.com/aliyun/fc-python-sdk)       |
| aliyun-python-sdk-ecs 4.10.1 | [云服务器SDK](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-ecs) |
| aliyun-python-sdk-vpc 3.0.2  | [专用网络SDK](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-vpc) |
| aliyun-python-sdk-rds 2.1.4  | [云数据库SDK](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-rds) |
| aliyun-python-sdk-kms 2.5.0  | [密钥管理服务SDK](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-kms) |
| pydatahub 2.11.2             | [DataHub SDK](https://github.com/aliyun/aliyun-datahub-sdk-python) |
| aliyun-mns 1.1.5             | [消息服务](https://github.com/gikoluo/aliyun-mns)            |
| aliyun-python-sdk-cdn 2.6.2  | [CDN服务](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-cdn) |
| aliyun-python-sdk-ram 3.0.0  | [访问控制RAM](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-ram) |
| aliyun-python-sdk-sts 3.0.0  | [访问控制STS](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-sts) |
| aliyun-python-sdk-iot 7.8.0  | [物联网平台IOT](https://github.com/aliyun/aliyun-openapi-python-sdk/tree/master/aliyun-python-sdk-iot) |
| aliyun-log-python-sdk 0.6.38 | [日志服务SLS](https://github.com/aliyun/aliyun-log-python-sdk) |

## 函数运行资源限制

[官网资源使用限制说明](https://help.aliyun.com/document_detail/51907.html?spm=a2c4g.11186623.6.817.53b75d5cqoKG02)

| 限制项                               | 资源上限（弹性实例） | 资源上限（性能实例） |
| ------------------------------------ | -------------------- | -------------------- |
| 临时磁盘空间（/tmp 空间）            | 512 MB               | 10 GB                |
| 文件描述符                           | 1024                 | 1024                 |
| 进程和线程总数                       | 1024                 | 1024                 |
| 函数最大申请内存                     | 3 GB                 | 16 GB                |
| 函数最大运行时间                     | 600s                 | 7200s                |
| Initializer最大运行时间              | 300s                 | 300s                 |
| 函数同步调用响应正文有效负载大小     | 6 MB                 | 6 MB                 |
| 函数异步调用请求正文有效负载大小     | 128 KB               | 128 KB               |
| 代码部署包大小（压缩为ZIP或JAR文件） | 100 MB               | 500 MB               |
| 原始代码大小                         | 500 MB               | 10 GB                |

## 对象测试事件模板

- 基本设置

```json
{
  "events": [
    {
      "eventName": "ObjectCreated:PutObject",
      "eventSource": "acs:oss",
      "eventTime": "2017-04-21T12:46:37.000Z",
      "eventVersion": "1.0",
      "oss": {
        "bucket": {
          "arn": "acs:oss:cn-hangzhou:1237050315505689:fc-jrtz",
          "name": "fc-jrtz",
          "ownerIdentity": "1237050315505689",
          "virtualBucket": ""
        },
        "object": {
          "deltaSize": 122539,
          "eTag": "688A7BF4F233DC9C88A80BF985AB7329",
          "key": "src-pk/000001_raw_data.xlsx",
          "size": 122539
        },
        "ossSchemaVersion": "1.0",
        "ruleId": "9adac8e253828f4f7c0466d941fa3db81161e853"
      },
      "region": "cn-hangzhou",
      "requestParameters": {
        "sourceIPAddress": "140.205.128.221"
      },
      "responseElements": {
        "requestId": "58F9FF2D3DF792092E12044C"
      },
      "userIdentity": {
        "principalId": "262561392693583141"
      }
    }
  ]
}
```

  