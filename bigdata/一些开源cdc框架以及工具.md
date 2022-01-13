## [一些开源cdc框架以及工具](https://www.cnblogs.com/rongfengliang/p/12058082.html)

以下是一些cdc工具，没有包含商业软件的

## zendesk maxwell

- 参考地址 
  <https://github.com/zendesk/maxwell>
- 功能 
  mysql 2 json 的kafaa 生产者

## airbnb SpinalTap

- 参考地址 
  <https://github.com/airbnb/SpinalTap>
- 功能 
  cdc 服务，捕获变动，并发送事件到下游，参考介绍<https://medium.com/airbnb-engineering/capturing-data-evolution-in-a-service-oriented-architecture-72f7c643ee6f>

## Yelp mysql_streamer

- 参考地址 
  <https://github.com/Yelp/mysql_streamer>
- 功能 
  mysql 的stream 服务，很可惜，很久没有更新了

## debezium debezium

- 参考地址 
  <https://github.com/debezium/debezium>
- 功能 
  cdc 服务，功能强大，支持的db 类型多，更新很频繁

## alibaba canal

- 参考资料 
  <https://github.com/alibaba/canal>
- 功能 
  基于mysql binlog 的数据订阅，消费组件，同样很强大

## alibaba otter

- 参考资料 
  <https://github.com/alibaba/otter>
- 功能 
  基于canal 解决数据同步的问题，同样很强大

## netflix dblog

- 参考资料 
  <https://medium.com/netflix-techblog/dblog-a-generic-change-data-capture-framework-69351fb9099b>
- 功能 
  cdc，但是从官方的介绍，超越cdc，目前还没开源，估计2020 会开源（待定，官方介绍）