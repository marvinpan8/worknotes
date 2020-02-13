# KONG consumer-services API
## 前言
- 域名 `XXXX.investoday.net`的最后一级子域名将与服务名一一对应，否则hmac认证将不通过
- name的命名规则为`consumer.username`$`service.name`

## GET 查询

- 获取全部列表： http://kongc.jrtzcloud.cn/consumer-services
- 获取全部列表： http://kongc.jrtzcloud.cn/consumer-services/
- 分页查询：http://kongc.jrtzcloud.cn/consumer-services?size=1
- 分页查询：http://kongc.jrtzcloud.cn/consumer-services?size=1&offset=XXXX,   offset为取以上返回值
- 获取单条记录： http://kongc.jrtzcloud.cn/consumer-services/:id_or_name
- 获取指定服务名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/services/:name/consumer-service
- 分页获取指定服务名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/services/:name/consumer-service?size=1
- 分页获取指定服务名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/services/:name/consumer-service?size=1&offset=XXXX,   offset为取以上返回值
- 获取指定用户名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/consumers/:username/consumer-service
- 分页获取指定用户名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/consumers/:username/consumer-service?size=1
- 分页获取指定用户名对应的用户服务关系列表：http://kongc.jrtzcloud.cn/consumers/:username/consumer-service?size=1&offset=XXXX,   offset为取以上返回值
- 获取指定服务名对应的指定用户服务关系单条记录：http://kongc.jrtzcloud.cn/services/:name/consumer-service/:id_or_name
- 获取指定用户名对应的指定用户服务关系单条记录：http://kongc.jrtzcloud.cn/consumers/:username/consumer-service/:id_or_name
- 获取指定用户服务关系id对应的consumer: http://kongc.jrtzcloud.cn/consumer-services/:id_or_name/consumer
- 获取指定用户服务关系id对应的service: http://kongc.jrtzcloud.cn/consumer-services/:id_or_name/service

> 注意：`endpoint_key`，当前设置为字段`name`, 若没有设置，默认取`primary_key`,主键一般为id
>

## POST 创建

- http://kongc.jrtzcloud.cn/consumer-services
- http://kongc.jrtzcloud.cn/services/:name/consumer-service  根据服务名创建，不传 service.id, 传了也无效
- http://kongc.jrtzcloud.cn/consumers/:username/consumer-service 根据用户名创建，不传 service.id, 传了也无效

> 注意事项：
>
> Content-Type:  application/json， created_at字段不传将自动创建。name的命名规则为`consumer.username`$`service.name`

```json
{
    "created_at": 1574843127,
	"consumer": {
        "id": "bd298542-382f-4512-bef1-c11c06909143"
    },
	"service": {
		"id": "319b8035-2ac5-4d90-b9ab-761ebd8d6715"
	},
    "name": "zhaoshang$blten",
	"expired_at": 1606360830
}
```

## DELETE 删除

- http://kongc.jrtzcloud.cn/consumer-services/:id_or_name
- http://kongc.jrtzcloud.cn/services/:name/consumer-service/:id_or_name
- http://kongc.jrtzcloud.cn/consumers/:username/consumer-service/:id_or_name

## PATCH 部分修改

- http://kongc.jrtzcloud.cn/consumer-services/:id_or_name
- http://kongc.jrtzcloud.cn/services/:name/consumer-service/:id_or_name
- http://kongc.jrtzcloud.cn/consumers/:username/consumer-service/:id_or_name

Content-Type:  application/json

```json
{
   "expired_at": 1606360840
}
```


## PUT全部修改，没有就创建

若路径参数为name, 没有就创建，且请求参数可以没有name；若路径参数为id, 则请求参数必须要有name
- http://kongc.jrtzcloud.cn/consumer-services/:id_or_name
- http://kongc.jrtzcloud.cn/services/:name/consumer-service/:id_or_name
- http://kongc.jrtzcloud.cn/consumers/:username/consumer-service/:id_or_name

Content-Type:  application/json

```json
{	
    "created_at": 1574843127,
	"consumer": {
        "id": "bd298542-382f-4512-bef1-c11c06909143"
    },
	"service": {
		"id": "319b8035-2ac5-4d90-b9ab-761ebd8d6715"
	},
    "name": "zhaoshang$blten",
	"expired_at": 1606360830
}
```



参考：<https://github.com/Kong/kong/blob/master/spec/03-plugins/19-hmac-auth/02-api_spec.lua>

## GET查询consumer

- 查询全部：http://kongc.jrtzcloud.cn/consumers
- 查询单个：http://kongc.jrtzcloud.cn/consumers/:id_or_username

## GET查询service

- 查询全部：http://kongc.jrtzcloud.cn/services

- 查询单个：http://kongc.jrtzcloud.cn/services/:id_or_name

  