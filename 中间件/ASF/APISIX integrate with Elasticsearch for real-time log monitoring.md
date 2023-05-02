## APISIX integrate with Elasticsearch for real-time log monitoring

> This article will introduce you to the relevant information of the elasticsearch-logger plugin of Apache APISIX, and report real-time logs of APISIX through the plugin.

### Background Information

Apache APISIX is a dynamic, real-time, high-performance API gateway that provides rich traffic management features such as load balancing, dynamic upstream, canary release, circuit breaking, authentication, observability, and more. It not only has many useful plugins, but also supports plugin dynamic change and hot reload.

Elasticsearch is a search engine based on the Lucene library. It provide a distributed, RESTful search and analytics engine capable of addressing a growing number of use cases. As the heart of the Elastic Stack, it centrally stores your data for lightning fast search, fine‑tuned relevancy, and powerful analytics that scale with ease.

### Introduction to the plugin

APISIX sends Runtime logs to Elasticsearch as HTTP requests. The plug-in elasticsearch-logger uses [bulk](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html#docs-bulk) format for log reporting, which allows APISIX to combine multiple logs and then report them, which makes the log reporting more flexible and has better performance. You can refer to the documentation [APISIX Batch Processor](https://apisix.apache.org/zh/docs/apisix/batch-processor/) for more detailed configuration of the log collection.

### How to Use

#### Prerequisites

This article is based on the following environments.

- OS: Centos 7.9
- Apache APISIX master branch, please refer to: [Apache APISIX Installation](https://apisix.apache.org/docs/apisix/installation-guide/)
- Elasticsearch 7.17.1
- Kibana 7.17.1

#### Step 1: Deploy Elasticsearch

The following uses `docker-compose` deploy Elasticsearch single node as an example. For other deployments, please refer to the [Elasticsearch official documentation](https://www.elastic.co/downloads/elasticsearch).

```shell
# Start 1 Elasticsearch node, 1 kibana with docker-compose
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.1
    container_name: elasticsearch
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      discovery.type: single-node
      xpack.security.enabled: 'false'
    networks:
      - es-net
    ports:
      - "9200:9200"
      - "9300:9300"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.1
    container_name: kibana
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
      I18N_LOCALE: zh-CN
    networks:
      - es-net
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"

networks:
  es-net:
    driver: bridge
```

#### Step 2: Create Routing and Enable Plugin

APISIX does not enable the `elasticsearch-logger` plugin by default. The following commands allow you to create routes and enable the `elasticsearch-logger` plugin.

```
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
            "ssl_verify":false,
            "retry_delay":1,
            "buffer_duration":60,
            "max_retry_count":0,
            "batch_max_size":1000,
            "inactive_timeout":5,
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

#### Step 3: Send Requests

Next, we send some requests through the API.

```shell
curl -i http://127.0.0.1:9080/elasticsearch.do\?q\=hello
HTTP/1.1 200 OK
...
hello, world
```

Now you can login the Kibana to view the log:

### Customize the log structure

Of course, we can also set the structure of the log data sent to Elasticsearch during use through the metadata configuration provided by the `elasticsearch-logger` plugin. By setting the `log_format` data, you can control the type of data sent.

For example, `$host` and `$time_iso8601` in the following data are built-in variables provided by Nginx; APISIX variables such as `$route_id` and `$service_id` are also supported.

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

A simple test by sending a request shows that the above logging structure settings have taken effect. Currently, Apache APISIX provides a variety of log format templates, which provides great flexibility in configuration, and more details on log format can be found in [Apache APISIX documentation](https://apisix.apache.org/docs/apisix/plugins/kafka-logger#metadata).

Now you can login the Kibana to view the custom log:



### Turn off the custom log structure

```shell
curl http://127.0.0.1:9180/apisix/admin/plugin_metadata/elasticsearch-logger \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X DELETE
```

Now, the `elasticsearch-logger` will use default structure to report the log.

### Turn off the plugin

If you are done using the plugin, simply remove the elasticsearch-logger plugin-related configuration from the route configuration and save it to turn off the plugin on the route. Thanks to the dynamic advantage of Apache APISIX, the process of turning on and off the plugin does not require restarting Apache APISIX, which is very convenient.

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

### Summary

In this article, we have introduced the features and usage steps of the `elasticsearch-logger` plugin. For more information about the elasticsearch-logger plugin and full configuration list, you can refer to the official documentation.

We are also currently working on other logging plugins to integrate with more related services. If you're interested in such integration projects, feel free to start a discussion in [GitHub Discussions](https://github.com/apache/apisix/discussions) or communicate via the [mailing list](https://apisix.apache.org/zh/docs/general/join).



### 参考资料

- https://apisix.apache.org/blog/2022/03/04/apigateway-clickhouse-makes-logging-easier/
- https://apisix.apache.org/blog/2022/01/17/apisix-kafka-integration/
- https://apisix.apache.org/blog/2021/12/13/monitor-apisix-ingress-controller-with-prometheus/

- https://apisix.apache.org/zh/blog/2022/07/13/monitor-api-gateway-apisix-with-prometheus/
- https://apisix.apache.org/zh/blog/2022/03/04/apigateway-clickhouse-makes-logging-easier/
- https://apisix.apache.org/zh/blog/2022/01/17/apisix-kafka-integration/