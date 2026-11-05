---
title: elasticsearch
date: 2026-03-11 13:48:00 +0800
categories: [三方插件, 数据库]  # categories是文章的分类，支持至多两级分类
tags: [elasticsearch, ES数据库]  # tags是文章的标签，可以有任意多个
---


注意：

elasticsearch与 es数据的大版本相关，注意测试功能是否正常

下文以elasticsearch8\.19\.3生成

## 1\. Elasticsearch 简介



Elasticsearch 是一个开源的分布式搜索和分析引擎，基于 Apache Lucene 构建。它提供了：

- **分布式实时文件存储**

- **全文检索引擎**

- **结构化数据搜索**

- **分析引擎**

### 核心概念



|概念|说明|类比关系型数据库|
|---|---|---|
|Index（索引）|文档的集合|Database（数据库）|
|Type（类型）|索引中的逻辑分类（ES 7\.x\+已废弃）|Table（表）|
|Document（文档）|JSON格式的数据记录|Row（行）|
|Field（字段）|文档中的属性|Column（列）|
|Mapping（映射）|字段类型定义|Schema（表结构）|
|Shard（分片）|索引的分片，用于分布式存储|\-|
|Replica（副本）|分片的备份，提高可用性|\-|

---

## 2\. 连接 Elasticsearch

### 2\.1 基础连接（无认证）



```Python
from elasticsearch import Elasticsearch

# 创建连接
es = Elasticsearch(
    ["http://localhost:9200"],
    # 超时设置
    request_timeout=30,
    # 重试次数
    max_retries=3,
    # 重试间隔
    retry_on_timeout=True
)

# 测试连接
if es.ping():
    print("连接成功！")
else:
    print("连接失败！")

# 查看集群信息
info = es.info()
print(info)
```



### 2\.2 带认证的连接



```Python
from elasticsearch import Elasticsearch

# 方式一：使用用户名密码
es = Elasticsearch(
    ["http://localhost:9200"],
    basic_auth=("elastic", "your_password"),
    request_timeout=30
)

# 方式二：使用 API Key
es = Elasticsearch(
    ["http://localhost:9200"],
    api_key="your_api_key_base64_encoded",
    request_timeout=30
)

# 方式三：使用 HTTP 头
es = Elasticsearch(
    ["http://localhost:9200"],
    headers={"Authorization": "ApiKey your_api_key"},
    request_timeout=30
)
```



### 2\.3 HTTPS 连接（带证书验证）



```Python
from elasticsearch import Elasticsearch

es = Elasticsearch(
    ["https://localhost:9200"],
    # 禁用证书验证（仅用于开发环境）
    verify_certs=False,
    # 或指定 CA 证书路径
    # ca_certs="/path/to/ca.crt",
    basic_auth=("elastic", "your_password")
)
```



### 2\.4 连接池配置



```Python
from elasticsearch import Elasticsearch

es = Elasticsearch(
    ["http://localhost:9200", "http://localhost:9201", "http://localhost:9202"],
    # 连接池大小
    maxsize=25,
    # 连接超时
    timeout=30,
    # 最大重试次数
    max_retries=3,
    # 重试状态码
    retry_on_status=(502, 503, 504),
    # 启用嗅探节点
    sniff_on_start=True,
    # 节点故障时重新嗅探
    sniff_on_connection_fail=True,
    # 定期嗅探节点
    sniffer_timeout=60
)
```



---



## 3\. 索引操作



### 3\.1 创建索引



#### 简单创建



```Python
from elasticsearch import Elasticsearch

es = Elasticsearch(["http://localhost:9200"])

# 创建索引（不指定映射）
index_name = "my_index"
if not es.indices.exists(index=index_name):
    response = es.indices.create(index=index_name)
    print(f"索引 {index_name} 创建成功")
    print(response)
```



#### 创建索引并指定映射



```Python
# 定义索引映射
mapping = {
    "mappings": {
        "properties": {
            "title": {
                "type": "text",
                "analyzer": "ik_max_word"  # 中文分词器
            },
            "content": {
                "type": "text",
                "analyzer": "ik_max_word"
            },
            "author": {
                "type": "keyword"  # 精确匹配
            },
            "publish_date": {
                "type": "date",
                "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
            },
            "views": {
                "type": "integer"
            },
            "tags": {
                "type": "keyword"
            },
            "location": {
                "type": "geo_point"  # 地理坐标
            }
        }
    },
    "settings": {
        "number_of_shards": 3,      # 主分片数
        "number_of_replicas": 1,    # 副本数
        "analysis": {
            "analyzer": {
                "ik_smart_analyzer": {
                    "type": "custom",
                    "tokenizer": "ik_smart"
                }
            }
        }
    }
}

# 创建索引
index_name = "articles"
response = es.indices.create(index=index_name, body=mapping)
print(f"索引 {index_name} 创建成功")
```



### 3\.2 查看索引



```Python
# 查看索引信息
index_info = es.indices.get(index="articles")
print(index_info)

# 查看索引映射
mapping = es.indices.get_mapping(index="articles")
print(mapping)

# 查看索引设置
settings = es.indices.get_settings(index="articles")
print(settings)

# 检查索引是否存在
exists = es.indices.exists(index="articles")
print(f"索引是否存在: {exists}")
```



### 3\.3 更新索引设置



```Python
# 更新副本数
settings_body = {
    "index": {
        "number_of_replicas": 2
    }
}
response = es.indices.put_settings(index="articles", body=settings_body)
print("设置更新成功")
```



### 3\.4 删除索引



```Python
# 删除单个索引
response = es.indices.delete(index="my_index")
print("索引删除成功")

# 删除多个索引
response = es.indices.delete(index=["index1", "index2"])

# 删除所有索引（谨慎使用！）
# response = es.indices.delete(index="*")
```



### 3\.5 索引别名



```Python
# 创建别名
response = es.indices.put_alias(
    index="articles",
    name="articles_alias"
)

# 查看别名
aliases = es.indices.get_alias(index="articles")
print(aliases)

# 删除别名
response = es.indices.delete_alias(
    index="articles",
    name="articles_alias"
)
```



---



## 4\. 文档操作



### 4\.1 创建文档



#### 指定 ID 创建



```Python
from elasticsearch import Elasticsearch
from datetime import datetime

es = Elasticsearch(["http://localhost:9200"])

# 创建文档（指定 ID）
doc = {
    "title": "Python Elasticsearch 教程",
    "content": "这是一篇关于 Python 使用 Elasticsearch 的详细教程",
    "author": "张三",
    "publish_date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    "views": 100,
    "tags": ["Python", "Elasticsearch", "教程"]
}

response = es.index(
    index="articles",
    id=1,  # 指定文档 ID
    document=doc
)

print(f"文档创建成功，ID: {response['_id']}")
print(response)
```



#### 自动生成 ID



```Python
# 创建文档（自动生成 ID）
doc = {
    "title": "Elasticsearch 高级查询",
    "content": "学习 Elasticsearch 的高级查询技巧",
    "author": "李四",
    "publish_date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    "views": 200,
    "tags": ["Elasticsearch", "查询"]
}

response = es.index(
    index="articles",
    document=doc  # 不指定 id，自动生成
)

print(f"文档创建成功，ID: {response['_id']}")
```



### 4\.2 查询文档



#### 根据 ID 查询



```Python
# 根据 ID 查询文档
response = es.get(index="articles", id=1)

if response['found']:
    print("文档内容:")
    print(response['_source'])
else:
    print("文档不存在")

# 检查文档是否存在
exists = es.exists(index="articles", id=1)
print(f"文档是否存在: {exists}")
```



#### 指定返回字段



```Python
# 只返回指定字段
response = es.get(
    index="articles",
    id=1,
    _source=["title", "author"]  # 只返回 title 和 author
)
print(response['_source'])
```



### 4\.3 更新文档



#### 部分更新



```Python
# 部分更新文档
update_body = {
    "doc": {
        "views": 150,
        "tags": ["Python", "Elasticsearch", "教程", "更新"]
    }
}

response = es.update(
    index="articles",
    id=1,
    body=update_body
)

print(f"文档更新成功，版本: {response['_version']}")
```



#### 使用脚本更新



```Python
# 使用脚本更新（增加浏览量）
update_body = {
    "script": {
        "source": "ctx._source.views += params.increment",
        "params": {
            "increment": 10
        }
    }
}

response = es.update(
    index="articles",
    id=1,
    body=update_body
)
print("浏览量已增加 10")
```



#### Upsert 操作（存在则更新，不存在则创建）



```Python
# Upsert 操作
update_body = {
    "doc": {
        "title": "更新后的标题",
        "views": 300
    },
    "doc_as_upsert": True  # 如果文档不存在，则创建
}

response = es.update(
    index="articles",
    id=999,
    body=update_body
)
print(response)
```



### 4\.4 删除文档



```Python
# 根据 ID 删除文档
response = es.delete(index="articles", id=1)
print(f"文档删除成功: {response['result']}")

# 删除前检查文档是否存在
if es.exists(index="articles", id=1):
    es.delete(index="articles", id=1)
    print("文档已删除")
else:
    print("文档不存在")
```



### 4\.5 批量操作



```Python
from elasticsearch.helpers import bulk

# 准备批量数据
actions = [
    {
        "_index": "articles",
        "_id": i,
        "_source": {
            "title": f"文章标题 {i}",
            "content": f"这是第 {i} 篇文章的内容",
            "author": "作者" + str(i % 5),
            "publish_date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "views": i * 10,
            "tags": ["Python", f"tag{i}"]
        }
    }
    for i in range(1, 101)
]

# 批量插入
success, failed = bulk(es, actions, stats_only=True)
print(f"成功: {success}, 失败: {failed}")
```



---



## 5\. 搜索操作



### 5\.1 基础搜索



#### 查询所有文档



```Python
# 查询所有文档
response = es.search(
    index="articles",
    body={
        "query": {
            "match_all": {}
        }
    }
)

print(f"总文档数: {response['hits']['total']['value']}")
for hit in response['hits']['hits']:
    print(hit['_source'])
```



#### 分页查询



```Python
# 分页查询
response = es.search(
    index="articles",
    body={
        "query": {
            "match_all": {}
        },
        "from": 0,      # 起始位置
        "size": 10,     # 每页大小
        "sort": [
            {"views": {"order": "desc"}}  # 按浏览量降序
        ]
    }
)

for hit in response['hits']['hits']:
    print(f"标题: {hit['_source']['title']}, 浏览量: {hit['_source']['views']}")
```



### 5\.2 全文搜索



#### Match 查询



```Python
# Match 查询（全文搜索）
response = es.search(
    index="articles",
    body={
        "query": {
            "match": {
                "content": "Python Elasticsearch"
            }
        }
    }
)

print(f"找到 {response['hits']['total']['value']} 条结果")
```



#### Match Phrase 查询（短语匹配）



```Python
# Match Phrase 查询（精确短语）
response = es.search(
    index="articles",
    body={
        "query": {
            "match_phrase": {
                "content": "Python 教程"
            }
        }
    }
)
```



#### Multi\-match 查询（多字段搜索）



```Python
# 多字段搜索
response = es.search(
    index="articles",
    body={
        "query": {
            "multi_match": {
                "query": "Python",
                "fields": ["title", "content"],
                "type": "best_fields"  # 最佳字段匹配
            }
        }
    }
)
```



### 5\.3 精确查询



#### Term 查询



```Python
# Term 查询（精确匹配，不分词）
response = es.search(
    index="articles",
    body={
        "query": {
            "term": {
                "author": "张三"
            }
        }
    }
)
```



#### Terms 查询（多值匹配）



```Python
# Terms 查询（匹配多个值）
response = es.search(
    index="articles",
    body={
        "query": {
            "terms": {
                "author": ["张三", "李四", "王五"]
            }
        }
    }
)
```



#### Range 查询（范围查询）



```Python
# Range 查询
response = es.search(
    index="articles",
    body={
        "query": {
            "range": {
                "views": {
                    "gte": 100,    # 大于等于
                    "lte": 500     # 小于等于
                }
            }
        }
    }
)

# 日期范围查询
response = es.search(
    index="articles",
    body={
        "query": {
            "range": {
                "publish_date": {
                    "gte": "2024-01-01",
                    "lte": "2024-12-31",
                    "format": "yyyy-MM-dd"
                }
            }
        }
    }
)
```



### 5\.4 复合查询



#### Bool 查询



```Python
# Bool 复合查询
response = es.search(
    index="articles",
    body={
        "query": {
            "bool": {
                "must": [  # 必须匹配
                    {
                        "match": {
                            "content": "Python"
                        }
                    }
                ],
                "should": [  # 应该匹配（加分）
                    {
                        "match": {
                            "title": "教程"
                        }
                    }
                ],
                "must_not": [  # 必须不匹配
                    {
                        "term": {
                            "author": "测试"
                        }
                    }
                ],
                "filter": [  # 过滤（不参与评分）
                    {
                        "range": {
                            "views": {
                                "gte": 100
                            }
                        }
                    }
                ]
            }
        }
    }
)
```



### 5\.5 聚合查询



```Python
# 聚合查询
response = es.search(
    index="articles",
    body={
        "size": 0,  # 不返回文档，只返回聚合结果
        "aggs": {
            "avg_views": {  # 平均浏览量
                "avg": {
                    "field": "views"
                }
            },
            "max_views": {  # 最大浏览量
                "max": {
                    "field": "views"
                }
            },
            "author_count": {  # 按作者分组统计
                "terms": {
                    "field": "author",
                    "size": 10
                },
                "aggs": {
                    "total_views": {
                        "sum": {
                            "field": "views"
                        }
                    }
                }
            }
        }
    }
)

print("聚合结果:")
print(f"平均浏览量: {response['aggregations']['avg_views']['value']}")
print(f"最大浏览量: {response['aggregations']['max_views']['value']}")
```



### 5\.6 高亮显示



```Python
# 搜索高亮
response = es.search(
    index="articles",
    body={
        "query": {
            "match": {
                "content": "Python"
            }
        },
        "highlight": {
            "fields": {
                "content": {}
            },
            "pre_tags": ["<em>"],
            "post_tags": ["</em>"],
            "fragment_size": 150
        }
    }
)

for hit in response['hits']['hits']:
    print(f"标题: {hit['_source']['title']}")
    if 'highlight' in hit:
        print(f"高亮: {hit['highlight']['content']}")
```

---



## 6\. 批量操作

### 6\.1 Bulk API



```Python
from elasticsearch.helpers import bulk
from datetime import datetime

# 批量操作示例
def generate_bulk_data():
    """生成批量操作数据"""
    actions = []
    
    # 批量创建
    for i in range(1, 101):
        actions.append({
            "_index": "articles",
            "_id": i,
            "_source": {
                "title": f"文章 {i}",
                "content": f"这是第 {i} 篇文章",
                "author": f"作者{i % 10}",
                "views": i * 10,
                "publish_date": datetime.now().strftime("%Y-%m-%d")
            }
        })
    
    # 批量更新
    for i in range(1, 51):
        actions.append({
            "_op_type": "update",
            "_index": "articles",
            "_id": i,
            "doc": {
                "views": i * 20
            }
        })
    
    # 批量删除
    for i in range(90, 101):
        actions.append({
            "_op_type": "delete",
            "_index": "articles",
            "_id": i
        })
    
    return actions

# 执行批量操作
actions = generate_bulk_data()
success, errors = bulk(es, actions, stats_only=False, raise_on_error=False)

print(f"成功: {success}")
if errors:
    print(f"错误: {errors}")
```



### 6\.2 批量查询



```Python
# 批量查询
response = es.mget(
    index="articles",
    body={
        "ids": [1, 2, 3, 4, 5]
    }
)

for doc in response['docs']:
    if doc['found']:
        print(f"ID: {doc['_id']}, 标题: {doc['_source']['title']}")
```



### 6\.3 Scroll API（大量数据遍历）



```Python
# 使用 Scroll API 遍历大量数据
response = es.search(
    index="articles",
    body={
        "query": {
            "match_all": {}
        },
        "size": 1000  # 每批大小
    },
    scroll="2m"  # 滚动上下文保持时间
)

scroll_id = response['_scroll_id']
total = response['hits']['total']['value']

print(f"总文档数: {total}")

# 处理第一批数据
for hit in response['hits']['hits']:
    print(hit['_source']['title'])

# 继续获取后续数据
while len(response['hits']['hits']) > 0:
    response = es.scroll(scroll_id=scroll_id, scroll="2m")
    scroll_id = response['_scroll_id']
    
    for hit in response['hits']['hits']:
        print(hit['_source']['title'])

# 清除滚动上下文
es.clear_scroll(scroll_id=scroll_id)
```



---



## 7\. 常见问题与最佳实践



### 7\.1 性能优化



#### 批量操作



```Python
# ❌ 不推荐：逐条插入
for doc in documents:
    es.index(index="my_index", document=doc)

# ✅ 推荐：批量插入
from elasticsearch.helpers import bulk
bulk(es, [{"_index": "my_index", "_source": doc} for doc in documents])
```



#### 使用 Filter 代替 Query



```Python
# ✅ Filter 不参与评分，性能更好
response = es.search(
    index="articles",
    body={
        "query": {
            "bool": {
                "filter": [  # 使用 filter
                    {"term": {"author": "张三"}},
                    {"range": {"views": {"gte": 100}}}
                ]
            }
        }
    }
)
```



#### 只返回需要的字段



```Python
# ✅ 只返回需要的字段
response = es.search(
    index="articles",
    body={
        "query": {"match_all": {}},
        "_source": ["title", "author"]  # 只返回 title 和 author
    }
)
```



### 7\.2 错误处理



```Python
from elasticsearch import Elasticsearch, NotFoundError, RequestError

es = Elasticsearch(["http://localhost:9200"])

try:
    # 尝试获取不存在的文档
    response = es.get(index="articles", id=999)
except NotFoundError:
    print("文档不存在")
except RequestError as e:
    print(f"请求错误: {e}")
except Exception as e:
    print(f"未知错误: {e}")
```



### 7\.3 连接管理



```Python
from elasticsearch import Elasticsearch
from contextlib import contextmanager

# 使用上下文管理器
@contextmanager
def get_es_client():
    es = Elasticsearch(["http://localhost:9200"])
    try:
        yield es
    finally:
        es.close()

# 使用
with get_es_client() as es:
    response = es.search(index="articles", body={"query": {"match_all": {}}})
    print(response)
```



### 7\.4 索引模板



```Python
# 创建索引模板
template = {
    "index_patterns": ["logs-*"],
    "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1
    },
    "mappings": {
        "properties": {
            "timestamp": {"type": "date"},
            "level": {"type": "keyword"},
            "message": {"type": "text"}
        }
    }
}

response = es.indices.put_template(name="logs_template", body=template)
print("索引模板创建成功")
```



### 7\.5 监控集群健康



```Python
# 查看集群健康状态
health = es.cluster.health()
print(f"集群状态: {health['status']}")
print(f"节点数: {health['number_of_nodes']}")
print(f"数据节点数: {health['number_of_data_nodes']}")

# 查看节点信息
nodes = es.nodes.info()
for node_id, node_info in nodes['nodes'].items():
    print(f"节点: {node_info['name']}, IP: {node_info['ip']}")

# 查看索引统计
stats = es.indices.stats(index="articles")
print(f"文档数: {stats['_all']['primaries']['docs']['count']}")
print(f"存储大小: {stats['_all']['primaries']['store']['size_in_bytes']} bytes")
```



---



## 总结



## 参考资源



- [Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/8.19/index.html)

- [Elasticsearch Python Client 官方文档](https://elasticsearch-py.readthedocs.io/)

- [Elasticsearch 中文文档](https://elasticsearch.bookhub.tech/)

- [菜鸟教程 \- Elasticsearch](https://www.cainiaojc.com/elasticsearch/)

