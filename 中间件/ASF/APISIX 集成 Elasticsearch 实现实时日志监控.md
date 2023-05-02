# APISIX 集成 Elasticsearch 实现实时日志监控

### 背景信息

**Apache APISIX** 是一个动态、实时、高性能的 **API** 网关，提供了负载均衡、动态上游、灰度发布、服务熔断、身份认证、可观测性等丰富的流量管理功能。作为 **API** 网关，**Apache APISIX** 不仅拥有丰富的插件，而且支持插件的热加载。

**Elasticsearch** 是一个基于 [Lucene](https://zh.m.wikipedia.org/zh-hans/Lucene) 库的搜索引擎。它提供了分布式、RESTful 风格的搜索和数据分析引擎，具有可扩展性，可分布式部署，可进行相关度搜索等特点，能够解决不断涌现出的各种用例。并且可以集中存储用户数据，帮助用户发现意料之中以及意料之外的情况。

### 插件介绍

**APISIX** 以 [bulk](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html#docs-bulk) 的格式，通过 **HTTP** 请求的方式向 **Elasticsearch** 发送 **APISIX** 的访问日志。

### 配置步骤

首先，你需要安装完成 **APISIX**，本文所有步骤基于 **Centos 7.5** 系统进行。你需要完成 **APISIX** 的安装，具体细节可参考 [APISIX 安装指南](https://apisix.apache.org/zh/docs/apisix/how-to-build/)。

#### 步骤1：启动 Elasticsearch

**APISIX** 默认不启用 `elasticsearch-logger` 插件。

本示例只演示了通过 `docker-compose` 启动 **Elasticsearch** 单节点的方式，其它启动方式可参考 [Elasticsearch 官方文档](https://www.elastic.co/cn/downloads/elasticsearch)。

```yaml
# 使用 docker-compose 启动 1 个 Elasticsearch 节点
# 账号密码为：elastic/123456
version: '3.8'
services:
  elasticsearch1:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.1
    restart: unless-stopped
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      discovery.type: single-node
      xpack.security.enabled: 'false'
      ELASTIC_USERNAME: elastic
      ELASTIC_PASSWORD: 123456
      xpack.security.enabled: 'true'
```

#### 步骤2：创建路由并开启插件

通过下方命令可进行路由创建与 `elasticsearch-logger` 插件的开启。

```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins":{
        "elasticsearch-logger":{
            "endpoint_addr":"http://127.0.0.1:9200",
            "field":{
                "index":"services",
                "type":"collector"
            },
            "auth":{
                "username":"elastic",
                "password":"123456"
            },
            "ssl_verify":false,
            "name":"elasticsearch-logger"
        }
    },
    "upstream":{
        "type":"roundrobin",
        "nodes":{
            "127.0.0.1:1980":1
        }
    },
    "uri":"/elasticsearch.do"
}'
```

上述代码中配置了 **Elasticsearch** 地址、目标 `field`，用户名与密码。

通过上述设置，就可以实现将 `/elasticsearch.do` 路径的 **API** 请求日志发送至 **Elasticsearch** 的功能。

#### 步骤3：发送请求

接下来我们通过 **API** 发送一些请求，并记录下请求次数。

```shell
curl -i http://127.0.0.1:9080/elasticsearch.do\?q\=hello
HTTP/1.1 200 OK
...
hello, world
```

通过 **HTTP** `GET` 请求即可从 **Elasticsearch** 中查看已经写入的日志：

```shell
curl -X GET "http://127.0.0.1:9200/services/_search" | jq .
{
  "took": 0,
   ...
    "hits": [
      {
        "_index": "services",
        "_type": "_doc",
        "_id": "M1qAxYIBRmRqWkmH4Wya",
        "_score": 1,
        "_source": {
          "apisix_latency": 0,
          "route_id": "1",
          "server": {
            "version": "2.15.0",
            "hostname": "apisix"
          },
          "request": {
            "size": 102,
            "uri": "/elasticsearch.do?q=hello",
            "querystring": {
              "q": "hello"
            },
            "headers": {
              "user-agent": "curl/7.29.0",
              "host": "127.0.0.1:9080",
              "accept": "*/*"
            },
            "url": "http://127.0.0.1:9080/elasticsearch.do?q=hello",
            "method": "GET"
          },
          "service_id": "",
          "latency": 0,
          "upstream": "127.0.0.1:1980",
          "upstream_latency": 1,
          "client_ip": "127.0.0.1",
          "start_time": 1661170929107,
          "response": {
            "size": 192,
            "headers": {
              "date": "Mon, 22 Aug 2022 12:22:09 GMT",
              "server": "APISIX/2.15.0",
              "content-type": "text/plain; charset=utf-8",
              "connection": "close",
              "transfer-encoding": "chunked"
            },
            "status": 200
          }
        }
      }
    ]
  }
}
```

### 自定义日志结构

当然，在使用过程中我们也可以通过 `elasticsearch-logger` 插件提供的元数据配置，来设置发送至 **Elasticsearch** 的日志数据结构。通过设置 `log_format` 数据，可以控制发送的数据类型。

比如以下数据中的 `$host`、`$time_iso8601` 等，都是来自于 **Nginx** 提供的内置变量；也支持如 `$route_id` 和 `$service_id` 等 **Apache APISIX** 提供的变量配置。

```shell
curl http://127.0.0.1:9180/apisix/admin/plugin_metadata/elasticsearch-logger \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "log_format": {
        "host": "$host",
        "@timestamp": "$time_iso8601",
        "client_ip": "$remote_addr"
    }
}'
```

通过发送请求进行简单测试，可以看到上述日志结构设置已生效。目前 **Apache APISIX** 提供多种日志格式模板，在配置上具有极大的灵活性，更多日志格式细节可参考 [Apache APISIX 官方文档](https://apisix.apache.org/docs/apisix/plugins/kafka-logger#metadata)。

此时通过  **HTTP** `GET` 请求查看写入 **Elasticsearch** 中的自定义日志：

```shell
curl -X GET "http://127.0.0.1:9200/services/_search" | jq .
{
  "took": 0,
  ...
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "services",
        "_type": "_doc",
        "_id": "NVqExYIBRmRqWkmH4WwG",
        "_score": 1,
        "_source": {
          "@timestamp": "2022-08-22T20:26:31+08:00",
          "client_ip": "127.0.0.1",
          "host": "127.0.0.1",
          "route_id": "1"
        }
      }
    ]
  }
}
```

### 关闭插件

如使用完毕，只需移除路由配置中 `elasticsearch-logger` 插件相关配置并保存，即可关闭路由上的插件。得益于 **Apache APISIX** 的动态化优势，开启关闭插件的过程都不需要重启 **Apache APISIX**，十分方便。

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/hello",
    "plugins": {},
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:1980": 1
        }
    }
}'
```

### 总结

本文为大家介绍了关于 **elasticsearch-logger** 插件的功能与使用步骤，更多关于 **elasticsearch-logger** 插件说明和完整配置列表，可以参考官方文档。

也欢迎随时在 [GitHub Discussions](https://github.com/apache/apisix/discussions) 中发起讨论，或通过[邮件列表](https://apisix.apache.org/zh/docs/general/join)进行交流。