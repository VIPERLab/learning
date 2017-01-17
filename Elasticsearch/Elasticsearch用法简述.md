# 前言
[Elasticsearch](https://www.elastic.co/products/elasticsearch)(以下简称ES)是目前比较流行的搜索引擎，除了常规的分词搜索功能，ES还提供了类SQL的结构化查询、LBS查询、模糊匹配等功能，中文分词方面有[medcl](https://github.com/medcl)提供的ES中文分词插件[ik](https://github.com/medcl/elasticsearch-analysis-ik)。大家如果同时有搜索加结构化查询的需求，推荐大家使用ES。

# 目录
* [与关系型数据的概念对比](#与关系型数据的概念对比)
* [官方推荐配置](#官方推荐配置)
* [索引和文档操作](#索引和文档操作)
  * [创建索引](#创建索引)
  * [重建索引](#重建索引)
  * [创建文档](#创建文档)
  * [合并段](#合并段)
* [结构化查询和搜索](#结构化查询和搜索)
  * [term查询](#term查询)
  * [range查询](#range查询)
  * [组合查询 ](#组合查询 )
  * [LBS查询](#lbs查询)
  * [搜索](#搜索)
  * [多字段搜索](#多字段搜索)
* [中文分词](#中文分词)
  * [测试分词效果](#测试分词效果)
  * [词库热更新](#词库热更新)
  
# 与关系型数据的概念对比
在Elasticsearch中，文档归属于一种类型(type),而这些类型存在于索引(index)中，我们可以画一些简单的对比图来类比传统关系型数据库：
> Relational DB -> Databases -> Tables -> Rows -> Columns

> Elasticsearch -> Indices   -> Types  -> Documents -> Fields 

Elasticsearch集群可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）。摘自[Elasticsearch权威指南（中文版）1.5](https://es.xiaoleilu.com/010_Intro/25_Tutorial_Indexing.html)。

[返回目录](#目录)

# 官方推荐配置
* open files
```bash
ulimit -n 65536
```

* max lock memory
```bash
ulimit -l unlimited
```

* max user processes
```bash
ulimit -u 2048
```

* max map count
```bash
sysctl -w vm.max_map_count = 262144
```

* memory_lock
```bash
# in elasticsearch.yml
bootstrap.memory_lock: true
```

* java opts
```bash
# Xms和Xmx相等, 不超过系统50%
export ES_JAVA_OPTS="-Xms16g -Xmx16g -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode";
```

官方推荐配置文档：[Important System Configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)

[返回目录](#目录)

# 索引和文档操作
## 创建索引
如创建一个campus索引，包含一个类型users：
```json
PUT /campus
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 4
  },
  "mappings": {
    "users": {
      "_all": {"enabled": false},
      "dynamic": "false",
      "properties": {
        "name":{
          "type":"string",
          "analyzer": "ik_max_word"
        },
        "tags":{
          "type":"keyword"
        },
        "school":{
          "properties":{
            "id": {"type": "integer"},
            "name": {
                "type": "string",
                "analyzer": "ik_max_word"
            }
          }
        },
        "gender":{
          "type":"byte"
        },
        "age":{
          "type":"integer"
        },
        "location": {
          "type":"geo_point"
        },
        "registertime": {
          "type":"long"
        }
      }
    }
  }
}

```

* [索引配置](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/index-modules.html)
  * number_of_replicas：副本数。
  * number_of_shards：分片数。[Elasticsearch权威指南（中文版）9](https://es.xiaoleilu.com/060_Distributed_Search/00_Intro.html)演示了分片数对查询或搜索的影响：
  ![查询阶段](https://es.xiaoleilu.com/images/elas_0901.png)
  ![取回阶段](https://es.xiaoleilu.com/images/elas_0902.png)

  我的理解是，设置大于1的分片数在查询或搜索时有并发的效果，但是会增加系统资源的使用。


* [映射配置](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/mapping.html)
  * [_all](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/mapping-all-field.html)：指定所有分词字段是否合并到_all字段，以供全文搜索。
  * [dynamic](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/dynamic.html)：在新建文档时，是否允许动态建立映射中没有的字段。


* [字段类型](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/mapping-types.html)
  * string类型的字段会被分词，建立倒排索引。
  * keyword类型的字段不会被分词，查询时需指定该字段的值。
  * geo_point类型用于LBS查询。

## 重建索引
ES中类型的字段支持新增，但不支持删除和修改。如需改变字段的类型或分词器，可以使用[reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/docs-reindex.html)。
如将campus索引的数据全部导入到new_campus索引：
```json
POST /_reindex
{
  "source": {
    "index": "campus"
  },
  "dest": {
    "index": "new_campus"
  }
}
```

## 创建文档
如在campus索引下新建一个类型为users的文档：
```json
PUT /campus/users/978906152
{ 
  "name": "哈哈",
  "school": {
    "name": "深圳大学",
    "id": 12023
  },
  "gender": 1,
  "registertime": 1473872047,
  "tags": [
    "行走的荷尔蒙",
    "英雄联盟"
  ],
  "age": 19,
  "location": {
    "lon": 113.93664550781,
    "lat": 22.543840408325
  }
}
```

## 合并段
ES中的文档每次提交都会新建，旧的文档会在合并段的过程中删除。可以调用[optimize API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-forcemerge.html)(5.1版本已更名为forcemerge)强制合并段。
```json
POST /campus/_optimize?max_num_segments=1
```

一般情况下不用调用optimize API，摘自[Elasticsearch权威指南（中文版）11.5](https://es.xiaoleilu.com/075_Inside_a_shard/60_Segment_merging.html)的警告：
> 不要在动态的索引（正在活跃更新）上使用optimize API。后台的合并处理已经做的很好了，优化命令会阻碍它的工作。不要干涉！

[返回目录](#目录)

# 结构化查询和搜索
[官方Query DSL文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

## term查询
如查询北京大学的所有用户：
```json
GET /campus/users/_search
{
  "query": {
    "term": {
      "school.name": "北京大学"
    }
  }
}

```

## range查询
如查询年龄在20-25岁的用户：
```json
GET /campus/users/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 20,
        "lte": 25
      }
    }
  }
}
```

## 组合查询 
如查询标签为“大长腿”或“小萝莉”的女性用户 ：
```json
GET /campus/users/_search
{
  "query": {
    "bool": {
      "filter": [{
        "terms": {
          "tags": ["大长腿", "小萝莉"]
        }
      }, {
        "term": {
          "gender": 2
        }
      }],
    }
  }
}
```

## LBS查询
如查询某个坐标点1km范围内的用户：
```json
GET /campus/users/_search
{
  "query":{
    "geo_distance": {
      "distance": "1km",
      "location":{
        "lat": 22.551002599147,
        "lon": 113.93987278436
      }
    }
  }
}
```

## 搜索
如搜索昵称中带“国强”的用户：
```json
GET /campus/users/_search
{
  "query":{
    "match":{
      "name":"国强"
    }
  }
}
```

## 多字段搜索
如想搜索昵称中带“国强”，在“北京”读书的用户，可以指定搜索词“国强 北京”，并且在name和school.name两个字段中搜索：
```json
GET /campus/users/_search
{
  "query":{
    "multi_match":{
      "query":"国强 北京",
      "type": "cross_fields",
      "fields": ["name^2", "school.name"],
      "operator": "and"
    }
  }
}
```

注意上面的搜索中如不指定`"operator": "and"`, 则对搜索词分词后，命中其中任何一个的文档都会做为搜索结果返回。

[返回目录](#目录)

# 中文分词

## 测试分词效果
可以在一个建好的索引上测试分词器的分词效果：
```json
GET /campus/_analyze?tokenizer=ik_max_word&text=中华人民共和国
```

## 词库热更新
ik的词库支持热更新，具体配置方法是：
```xml
<!-- in /your-root-to-es/plugins/ik/config/IKAnalyzer.cfg.xml -->
<properties>
    <!--用户可以在这里配置远程扩展字典 -->
    <entry key="remote_ext_dict">http://yoursite.com/getCustomDict</entry>
    <!--用户可以在这里配置远程扩展停止词字典-->
    <entry key="remote_ext_stopwords">http://xxx.com/xxx.dic</entry>
</properties>

```

网上可以找到一些流行词库，如搜狗每周发布的[网络流行新词](http://pinyin.sogou.com/dict/detail/index/4?rf=dictindex)。另有大神提供的[搜狗词库转换文本的工具](https://github.com/aboutstudy/scel2mmseg)。然后将转换后的文本放到你的一个nginx上即可。

[返回目录](#目录)
