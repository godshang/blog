---
title: 'ElasticSearch之一'
layout: post
categories: 技术
tags:
    - ElasticSearch
---

# 安装

以Linux环境为例，从官网下载rpm包，这里下载到的是6.5.0版本，可以选择最新的版本下载。

执行以下命令进行安装：

```
rpm -ivh elasticsearch-6.5.0.rpm

```

显示以下信息：

```
[root@spiderman-14f9e6f5a data]# rpm -ivh elasticsearch-6.5.0.rpm 
warning: elasticsearch-6.5.0.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Preparing...                          ################################# [100%]
Creating elasticsearch group... OK
Creating elasticsearch user... OK
Updating / installing...
   1:elasticsearch-0:6.5.0-1          ################################# [100%]
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service
Created elasticsearch keystore in /etc/elasticsearch
```

我们按照提示依次执行命令，完成ES的安装。

下面检查以下ES是否正常运行：

```
curl -XGET 'localhost:9200/?pretty'
```

如果得到如下的信息，则说明已经启动成功。

```
{
  "name" : "llvd8Ye",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "9U8q7ZjxT1-zOxsnpHYJXQ",
  "version" : {
    "number" : "6.5.0",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "816e6f6",
    "build_date" : "2018-11-09T18:58:36.352602Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

# 常用命令

## 关闭ES

```
curl -XPOST 'http://localhost:9200/_shutdown'
```

## 创建文档

创建一个文档， megacorp 为索引名， employee 为 type 名称， 1 为 文档的 ID ：

```
curl -H "Content-Type:application/json" -X PUT http://localhost:9200/megacorp/employee/1 -d '
{
  "first_name": "John",
  "last_name": "Smith",
  "age": 25,
  "about": "I love to go rock climbing",
  "interests": [ "sports", "music" ]
}
'
```

## 查询

按照 ID 查询：

```
curl -X GET http://localhost:9200/megacorp/employee/1?pretty
```

search 查询：

```
curl -X GET http://localhost:9200/megacorp/employee/_search -H 'Content-Type: application/json' -d'
{
  "query": { 
    "match": {
      "last_name": "Smith"
    }
  }
}
'
```

更复杂一些的查询：

```
curl -X GET http://localhost:9200/megacorp/employee/_search -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "last_name": "smith"
        }
      },
      "filter": {
        "range": {
          "age": {
            "gt": 30
          }
        }
      }
    }
  }
}
'
```

Phrase 查询

```
curl -X GET http://localhost:9200/megacorp/employee/_search -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": {
        "match_phrase": {
          "about": "rock climbing"
        }
      }
    }
  }
}
'
```

高亮查询

```
curl -X GET http://localhost:9200/megacorp/employee/_search -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": {
        "match_phrase": {
          "about": "rock climbing"
        }
      }
    }
  },
  "highlight": {
    "fields": {
      "about": {}
    }
  }
}
'
```

聚合查询

```
curl -X GET http://localhost:9200/megacorp/employee/_search -H 'Content-Type: application/json' -d'
{
  "aggs":{
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
'
```



## 删除索引

```
curl -X DELETE http://localhost:9200/megacorp
```

## 删除文档

删除 query 匹配到的数据：

```
curl -X POST http://localhost:9200/megacorp/_delete_by_query -H 'Content-Type: application/json' -d'
{
  "query": { 
    "match": {
      "last_name": "Smith"
    }
  }
}
'
```