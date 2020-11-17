Elasticsearch 备忘录

##### 测试环境安装完es之后，第一件事应该是安装一个可视化操作客户端，如:[elasticsearch-head，(转载)](http://note.youdao.com/s/XCI8o1JD)

#### 索引操作

##### 创建索引（使用分词）
创建一个只有一个字段的索引test，名称name，类型text
```json
curl -XPUT 'localhost:9200/test' -H 'content-Type:application/json' -d '
{
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
        "name" : {
            "type" : "text",
            "analyzer" : "index_ansj",
            "search_analyzer" : "query_ansj"
        }
    }
  }
}'
```

##### 删除索引
```
curl -XDELETE 'localhost:9200/test'
```

##### 创建数据
```json
curl -XPOST localhost:9200/test/_doc -H 'Content-Type: application/json' -d '{
    "name" : "中国人民万岁",
    "post_date" : "2020-11-11 11:11:11",
    "message" : "trying out Elasticsearch"
}'
```

##### head里其他概念
```textmate
term           精确匹配输入的参数（查询条件不使用分析器）
wildcard       包含与通配符模式匹配的术语的文档(可使用正则匹配）
prefix         前缀匹配
fuzzy          模糊匹配 
range          范围匹配，匹配数值类型
query_string   使用具有严格语法的分析器，根据提供的查询字符串返回文档
text           全文搜索，目前我使用这个查询会报错
missing        是否存在该字段值，和exists反义
```

#### 相关问题，[参考](https://www.cnblogs.com/duanxz/p/3508338.html)

###### match和term的区别     
1. term  
    1.1) term查询keyword字段，term不会分词。而keyword字段也不分词。需要完全匹配才可。
    
    2.2) term查询text字段，因为text字段会分词，而term不分词，所以term查询的条件必须是text字段分词后的某一个。

2. match  
    2.1) match查询keyword字段，match会被分词，而keyword不会被分词，match的需要跟keyword的完全匹配可以。其他的不完全匹配的都是失败的。
    
    2.2) match查询text字段，match分词，text也分词，只要match的分词结果和text的分词结果有相同的就匹配。

3. match_phrase  
    3.1) match_phrase匹配keyword字段。这个同上必须跟keywork一致才可以。

    3.2) match_phrase匹配text字段。match_phrase是分词的，text也是分词的。match_phrase的分词结果必须在text字段分词中都包含，而且顺序必须相同，而且必须都是连续的。

4. query_string  
    4.1) query_string查询keyword类型的字段，试过了，无法查询。

    4.2) query_string查询text类型的字段。和match_phrase区别的是，不需要连续，顺序还可以调换。
