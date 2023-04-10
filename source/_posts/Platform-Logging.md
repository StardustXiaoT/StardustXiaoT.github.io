---
title: Platform Logging
tags:
  - Config
categories:
  - Configuration
date: 2023-04-10 15:15:40
cover: https://images.unsplash.com/photo-1571035089306-e40c9289fe78?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=687&q=80
---


For Rancher 2.6, refers this article：

{% blockquote 袁振    https://zhuanlan.zhihu.com/p/481258609 知乎链接 %}
Rancher 2.6 全新 Logging 快速入门
{% endblockquote %}


For Rancher 2.5 and 2.5 only, please refer below steps.

## Abstract
在 SUSE Rancher 2.5 以前，日志收集的架构是使用 Fluentd 收集指定目录日志或者容器标准输出流后，再通过 UI 日志功能页面的配置发送到指定的后端。如 Elasticsearch、splunk、kafka、syslog、fluentd 等常见的后端工具，可以自动化地收集 kubernetes 集群和运行于集群之上的工作负载日志。这种方式无疑是非常简单、易用、方便的，但是随着用户对云原生理解的加深，对业务日志分析颗粒度要求的提高，之前的日志收集方式有些太过死板，灵活性非常差，已经无法满足对日志收集具有更高要求的用户了。

于是从 SUSE Rancher 2.5 版本开始，BanzaiCloud 公司的开源产品 Logging Operator 成为了新一代 SUSE Rancher 容器云平台日志采集功能的组成组件。SUSE Rancher 2.5 版本作为新日志组件的过渡版本，保留了老版本的日志收集方式，并将全新的 Logging Operator 作为实验性功能使用；SUSE Rancher 2.6 版本则已经完全使用了 Logging Operator 作为日志收集工具

## Installation

从 Rancher Market Place 中安装Logging 模块， 里面包括了logging-operator， fluent/fluentbit 组件。 

从Rancher Market中安装EFK（其他处理日志软件如Loki也是可以的， 取决于项目需求）。

## Basic flow
Fluentbit会采集container中log里的内容，发动给fluentd。 然后fluentd会根据配置把日志发给不同的目的地。

## Configuration

banzai cloud 在自定义资源中主要给用户提供了两个重要的资源， flow 和 output。

在Rancher 2.5中， 需要自己定义这两个资源。 clusterflow ， clusterouput与 flow, output的区别是作用域不同。



ClusterOutput 内容是fluentbit的插件定义， 这里启用了elasticsearch的插件并添加了目的地等参数。
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: es-clusteroutput
spec:
  elasticsearch:
    host: elasticsearch-master.efk.svc.cluster.local
    port: 9200
    scheme: http
    index_name: "fluentd.${tag}.%Y%m%d"
```

Clusterflow  此为fluentd的插件描述， 这边定义了获取源的select以及output的去向。 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: clusterflow
spec:
  match:
  - select:
      namespaces:
      - mingxin-prod
      - mingxin-test
  filters:
    - dedot:
        de_dot_separator: "-"
        de_dot_nested: true
#    - stdout:
#       output_type: json
  globalOutputRefs:
    - es-clusteroutput
```
以上资源需要在cattle-logging-system中应用。 

## Post action
由于使用老旧版本的EFK， 7.3.0需要手动做一些工作。 

在EFK接受到之后，需要在Kibana上建立index， 并且使用api建立index template。
PUT /_template/fluentd
```json
{
  "index_patterns": ["fluentd.*"],
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "_source": {
      "enabled": true
    },
    "properties": {
      "host_name": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date",
        "format": "EEE MMM dd HH:mm:ss Z yyyy"
      }
    }
  }
}
```