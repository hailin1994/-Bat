## 十二、深入聚合数据分析

### 12.1 bucket与metric两个核心概念

bucket：就是对数据进行分组，类似MySQL中的group

metric：对一个数据分组执行的统计；metric就是对一个bucket执行的某种聚合分析的操作，比如说求平均值，求最大值，求最小值

### 12.2 家电卖场案例以及统计哪种颜色电视销量最高

以一个家电卖场中的电视销售数据为背景，来对各种品牌，各种颜色的电视的销量和销售额，进行各种各样角度的分析

#### 初始化数据

```bash
PUT /tvs
{
    "mappings": {
        "properties": {
            "price": {
                "type": "long"
            },
            "color": {
                "type": "keyword"
            },
            "brand": {
                "type": "keyword"
            },
            "sold_date": {
                "type": "date"
            }
        }
    }
}
```

添加数据

```bash
POST /tvs/_bulk
{ "index": {}}
{ "price" : 1000, "color" : "红色", "brand" : "长虹", "sold_date" : "2019-10-28" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2019-11-05" }
{ "index": {}}
{ "price" : 3000, "color" : "绿色", "brand" : "小米", "sold_date" : "2019-05-18" }
{ "index": {}}
{ "price" : 1500, "color" : "蓝色", "brand" : "TCL", "sold_date" : "2019-07-02" }
{ "index": {}}
{ "price" : 1200, "color" : "绿色", "brand" : "TCL", "sold_date" : "2019-08-19" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2019-11-05" }
{ "index": {}}
{ "price" : 8000, "color" : "红色", "brand" : "三星", "sold_date" : "2020-01-01" }
{ "index": {}}
{ "price" : 2500, "color" : "蓝色", "brand" : "小米", "sold_date" : "2020-02-12" }
```

#### 统计哪种颜色的电视销量最高

```bash
GET /tvs/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            }
        }
    }
}
```

> size：只获取聚合结果，而不要执行聚合的原始数据
>  aggs：固定语法，要对一份数据执行分组聚合操作
>  popular_colors：就是对每个aggs，都要起一个名字，这个名字是随机的，你随便取什么都ok
>  terms：根据字段的值进行分组
>  field：根据指定的字段的值进行分组

返回结果说明：

- hits.hits：我们指定了size是0，所以hits.hits就是空的，否则会把执行聚合的那些原始数据给你返回回来
- aggregations：聚合结果
- popular_color：我们指定的某个聚合的名称
- buckets：根据我们指定的field划分出的buckets
- key：每个bucket对应的那个值
- doc_count：这个bucket分组内，有多少个数据
- 数量，其实就是这种颜色的销量

每种颜色对应的bucket中的数据的默认的排序规则：按照doc_count降序排序

### 12.3 实战bucket+metric：统计每种颜色电视平均价格

```bash
GET /tvs/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": { 
            "avg_price": { 
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}    
```

### 12.4 bucket嵌套实现颜色+品牌的多层下钻分析

统计每个颜色的平均价格，同时统计每个颜色下每个品牌的平均价格

```bash
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "color_avg_price": {
          "avg": {
            "field": "price"
          }
        },
        "group_by_brand": {
          "terms": {
            "field": "brand"
          },
          "aggs": {
            "brand_avg_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```

这里需要知道的是es是根据语句顺序执行的，就像人去读取执行一样。

### 12.5 掌握更多metrics：统计每种颜色电视最大最小价格

更多的metric

```swift
count：bucket，terms，自动就会有一个doc_count，就相当于是count
avg：avg aggs，求平均值
max：求一个bucket内，指定field值最大的那个数据
min：求一个bucket内，指定field值最小的那个数据
sum：求一个bucket内，指定field值的总和

GET /tvs/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { "avg": { "field": "price" } },
            "min_price" : { "min": { "field": "price"} }, 
            "max_price" : { "max": { "field": "price"} },
            "sum_price" : { "sum": { "field": "price" } } 
         }
      }
   }
}
```

### 12.6 实战histogram按价格区间统计电视销量和销售额

histogram：类似于terms，也是进行bucket分组操作，接收一个field，按照这个field的值的各个范围区间，进行bucket分组操作；比如：

```bash
"histogram":{ 
  "field": "price",
  "interval": 2000
},
```

interval：2000，划分范围，02000，20004000，40006000，60008000，8000~10000分组

根据price的值，比如2500，看落在哪个区间内，比如20004000，此时就会将这条数据放入20004000对应的那个bucket中

bucket划分的方法，terms，将field值相同的数据划分到一个bucket中；bucket有了之后，去对每个bucket执行avg，count，sum，max，min，等各种metric聚合分析操作

```bash
GET /tvs/_search
{
   "size" : 0,
   "aggs":{
      "price":{
         "histogram":{ 
            "field": "price",
            "interval": 2000
         },
         "aggs":{
            "revenue": {
               "sum": { 
                 "field" : "price"
               }
             }
         }
      }
   }
}
```

### 12.7 实战date histogram之统计每月电视销量

- histogram，按照某个值指定的interval划分；
- date histogram，按照我们指定的某个date类型的日期field，以及日期interval，按照一定的日期间隔，去划分；

```bash
GET /tvs/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold_date",
            "interval": "month", 
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0, 
            "extended_bounds" : { 
                "min" : "2019-01-01",
                "max" : "2020-12-31"
            }
         }
      }
   }
}
```

min_doc_count：即使某个日期interval，2019-01-01~2019-01-31中，一条数据都没有，那么这个区间也是要返回的，不然默认是会过滤掉这个区间的

extended_bounds：min，max：划分bucket的时候，会限定在这个起始日期，和截止日期内

### 12.8 下钻分析之统计每季度每个品牌的销售额

```bash
GET /tvs/_search
{
  "size": 0,
  "aggs": {
    "group_by_sold_date": {
      "date_histogram": {
        "field": "sold_date",
        "interval": "quarter",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2016-01-01",
          "max": "2017-12-31"
        }
      },
      "aggs": {
        "group_by_brand": {
          "terms": {
            "field": "brand"
          },
          "aggs": {
            "sum_price": {
              "sum": {
                "field": "price"
              }
            }
          }
        },
        "total_sum_price": {
          "sum": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 12.9 搜索+聚合：统计指定品牌下每个颜色的销量

```bash
GET /tvs/_search 
{
  "size": 0,
  "query": {
    "term": {
      "brand": {
        "value": "小米"
      }
    }
  },
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      }
    }
  }
}
```

es的任何的聚合，都必须在搜索出来的结果数据中进行聚合分析操作。

### 12.10 global bucket：单个品牌与所有品牌销量对比

一个聚合操作，必须在query的搜索结果范围内执行

上面的需求需要出来两个结果，一个结果，是基于query搜索结果来聚合的; 一个结果，是对所有数据执行聚合的

```bash
GET /tvs/_search 
{
  "size": 0, 
  "query": {
    "term": {
      "brand": {
        "value": "长虹"
      }
    }
  },
  "aggs": {
    "single_brand_avg_price": {
      "avg": {
        "field": "price"
      }
    },
    "all": {
      "global": {},
      "aggs": {
        "all_brand_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

global：就是global bucket，就是将所有数据纳入聚合的scope，而不管之前的query

### 12.11 过滤+聚合：统计价格大于1200的电视平均价格

搜索+聚合
 过滤+聚合

```bash
GET /tvs/_search 
{
  "size": 0,
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "price": {
            "gte": 1200
          }
        }
      }
    }
  },
  "aggs": {
    "avg_price": {
      "avg": {
        "field": "price"
      }
    }
  }
}
```

### 12.12 bucket filter：统计牌品最近一个月的平均价格

```bash
GET /tvs/_search 
{
  "size": 0,
  "query": {
    "term": {
      "brand": {
        "value": "长虹"
      }
    }
  },
  "aggs": {
    "recent_150d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-150d"
          }
        }
      },
      "aggs": {
        "recent_150d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "recent_140d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-140d"
          }
        }
      },
      "aggs": {
        "recent_140d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "recent_130d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-130d"
          }
        }
      },
      "aggs": {
        "recent_130d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

bucket filter：对不同的bucket下的aggs，进行filter

### 12.13 排序：按每种颜色的平均销售额降序排序

默认排序，是按照每个bucket的doc_count降序来排的

```bash
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

指定排序规则

```bash
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color",
        "order": {
          "avg_price": "asc"
        }
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 12.14 颜色+品牌下钻分析时按最深层metric进行排序

```bash
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"},
      "aggs": {
        "group_by_brand": {
          "terms": {
            "field": "brand",
            "order": {
              "avg_price": "desc"
            }
          },
          "aggs": {
            "avg_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```

### 12.15 易并行聚合算法，三角选择原则，近似聚合算法

#### 易并行聚合算法：max

有些聚合分析的算法，是很容易就可以并行的，比如说max，只需要各个节点单独求最大，然后将结果返回再求最大值即可。

有些聚合分析的算法，是不好并行的，比如count(distinct)，并不是在每个node上，直接就去重求和就可以的，因为数据可能会很多，同时各个节点之间也有重复数据的情况；

因此为提高性能es会采取近似聚合的方式，就是采用在每个node上进行近估计的方式，得到最终的结论；
 近似估计后的结果，不完全准确，但是速度会很快，一般会达到完全精准的算法的性能的数十倍

#### 三角选择原则

精准+实时+大数据 --> 选择2个

- （1）精准+实时: 没有大数据，数据量很小，那么一般就是单机跑，随便你则么玩儿就可以
- （2）精准+大数据：hadoop，批处理，非实时，可以处理海量数据，保证精准，可能会跑几个小时
- （3）大数据+实时：es，不精准，近似估计，可能会有百分之几的错误率

#### 近似聚合算法

如果采取近似估计的算法：延时在100ms左右，0.5%错误

如果采取100%精准的算法：延时一般在5s~几十s，甚至几十分钟、几个小时， 0%错误

### 12.16 cardinality去重算法以及每月销售品牌数量统计

cartinality metric：对每个bucket中的指定的field进行去重，取去重后的count，类似于count(distcint)

```bash
GET /tvs/_search
{
  "size" : 0,
  "aggs" : {
      "months" : {
        "date_histogram": {
          "field": "sold_date",
          "interval": "month"
        },
        "aggs": {
          "distinct_colors" : {
              "cardinality" : {
                "field" : "brand"
              }
          }
        }
      }
  }
}
```

### 12.17 cardinality算法之优化内存开销以及HLL算法

cardinality，count(distinct)，5%的错误率，性能在100ms左右

#### precision_threshold优化准确率和内存开销

```bash
GET /tvs/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_brand" : {
            "cardinality" : {
              "field" : "brand",
              "precision_threshold" : 100 
            }
        }
    }
}
```

brand去重，如果brand的unique value在precision_threshold个以内，cardinality，几乎保证100%准确

cardinality算法，会占用precision_threshold * 8 byte 内存消耗，100 * 8 = 800个字节；
 占用内存很小,而且unique value如果的确在值以内，那么可以确保100%准确；数百万的unique value，错误率在5%以内

precision_threshold，值设置的越大，占用内存越大，1000 * 8 = 8000 / 1000 = 8KB，可以确保更多unique value的场景下，100%的准确

#### HyperLogLog++ (HLL)算法性能优化

cardinality底层算法：HLL算法，HLL算法的性能

对所有的uqniue value取hash值，通过hash值近似去求distcint count，存在误差

默认情况下，发送一个cardinality请求的时候，会动态地对所有的field value，取hash值; 将取hash值的操作，前移到建立索引的时候

构建hash

```bash
PUT /tvs
{
  "mappings": {
      "properties": {
        "brand": {
          "type": "text",
          "fields": {
            "hash": {
              "type": "murmur3" 
            }
          }
        }
      }
    }
}
```

基于hash进行去重查询

```bash
GET /tvs/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_brand" : {
            "cardinality" : {
              "field" : "brand.hash",
              "precision_threshold" : 100 
            }
        }
    }
}
```

### 12.18 percentiles百分比算法以及网站访问时延统计

需求：比如有一个网站，记录下了每次请求的访问的耗时，需要统计tp50，tp90，tp99

- tp50：50%的请求的耗时最长在多长时间
- tp90：90%的请求的耗时最长在多长时间
- tp99：99%的请求的耗时最长在多长时间

设置mapping

```bash
PUT /website
{
    "mappings": {
      "properties": {
          "latency": {
              "type": "long"
          },
          "province": {
              "type": "keyword"
          },
          "timestamp": {
              "type": "date"
          }
      }
    }
}
```

添加数据

```bash
POST /website/_bulk
{ "index": {}}
{ "latency" : 105, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 83, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 92, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 112, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 68, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 76, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 101, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 275, "province" : "新疆", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 166, "province" : "新疆", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 654, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 389, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 302, "province" : "新疆", "timestamp" : "2016-10-29" }
```

统计数据

```bash
GET /website/_search 
{
  "size": 0,
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "latency",
        "percents": [
          50,
          95,
          99
        ]
      }
    },
    "latency_avg": {
      "avg": {
        "field": "latency"
      }
    }
  }
}
```

50%的请求，数值的最大的值是多少，不是完全准确的

```bash
GET /website/_search 
{
  "size": 0,
  "aggs": {
    "group_by_province": {
      "terms": {
        "field": "province"
      },
      "aggs": {
        "latency_percentiles": {
          "percentiles": {
            "field": "latency",
            "percents": [
              50,
              95,
              99
            ]
          }
        },
        "latency_avg": {
          "avg": {
            "field": "latency"
          }
        }
      }
    }
}
```

### 12.19 percentiles rank以及网站访问时延SLA统计

SLA：就是你提供的服务的标准

我们的网站的提供的访问延时的SLA，确保所有的请求100%，都必须在200ms以内，大公司内，一般都是要求100%在200ms以内

如果超过1s，则需要升级到A级故障，代表网站的访问性能和用户体验急剧下降

需求：在200ms以内的，有百分之多少，在1000毫秒以内的有百分之多少，percentile ranks metric

这个percentile ranks，其实比pencentile还要常用

按照品牌分组，计算，电视机，售价在1000占比，2000占比，3000占比



```bash
GET /website/_search 
{
  "size": 0,
  "aggs": {
    "group_by_province": {
      "terms": {
        "field": "province"
      },
      "aggs": {
        "latency_percentile_ranks": {
          "percentile_ranks": {
            "field": "latency",
            "values": [
              200,
              1000
            ]
          }
        }
      }
    }
  }
}
```

percentile的优化：TDigest算法，用很多的节点来执行百分比的计算，近似估计，有误差，节点越多，越精准

compression：限制节点数量最多 compression * 20 = 2000个node去计算；默认100；越大，占用内存越多，越精准，性能越差

一个节点占用32字节，100 * 20 * 32 = 64KB

如果你想要percentile算法越精准，compression可以设置的越大

### 12.20 基于doc value正排索引的聚合内部原理

- 聚合分析的内部原理是什么？
- aggs，term，metric avg max这些执行一个聚合操作的时候，内部原理是怎样的呢？
- 用了什么样的数据结构去执行聚合？是不是用的倒排索引？

### 12.21 doc value机制内核级原理深入探秘

#### doc value原理

- （1）index-time生成

PUT/POST的时候，就会生成doc value数据，也就是正排索引

- （2）核心原理与倒排索引类似

正排索引，也会写入磁盘文件中，然后呢，os cache先进行缓存，以提升访问doc value正排索引的性能
 如果os cache内存大小不足够放得下整个正排索引，doc value，就会将doc value的数据写入磁盘文件中

- （3）性能问题：给jvm更少内存，64g服务器，给jvm最多16g

es官方是建议，es大量是基于os cache来进行缓存和提升性能的，不建议用jvm内存来进行缓存，那样会导致一定的gc开销和oom问题；
 给jvm更少的内存，给os cache更大的内存；64g服务器，给jvm最多16g，几十个g的内存给os cache；
 os cache可以提升doc value和倒排索引的缓存和查询效率

#### column压缩

```undefined
doc1: 550
doc2: 550
doc3: 500
```

合并相同值，550，doc1和doc2都保留一个550的标识即可

```undefined
（1）所有值相同，直接保留单值
（2）少于256个值，使用table encoding模式：一种压缩方式
（3）大于256个值，看有没有最大公约数，有就除以最大公约数，然后保留这个最大公约数

doc1: 36
doc2: 24
```

6 --> doc1: 6, doc2: 4 --> 保留一个最大公约数6的标识，6也保存起来

如果没有最大公约数，采取offset结合压缩的方式：

#### disable doc value

如果的确不需要doc value，比如聚合等操作，那么可以禁用，减少磁盘空间占用

```bash
PUT /my_index
{
  "mappings": {
      "properties": {
        "my_field": {
          "type":       "keyword"
          "doc_values": false 
        }
      }
    }
}
```

### 12.22 string field聚合实验以及fielddata原理初探

#### 对于分词的field执行aggregation，发现报错

```bash
GET /test_index/_search 
{
  "aggs": {
    "group_by_test_field": {
      "terms": {
        "field": "test_field"
      }
    }
  }
}
```

对分词的field，直接执行聚合操作，会报错，大概意思是说，你必须要打开fielddata，然后将正排索引数据加载到内存中，才可以对分词的field执行聚合操作，而且会消耗很大的内存

#### 给分词的field，设置fielddata=true，发现可以执行，但是结果似乎不是我们需要的

如果要对分词的field执行聚合操作，必须将fielddata设置为true

```bash
POST /test_index/_mapping
{
  "properties": {
    "test_field": {
      "type": "text",
      "fielddata": true
    }
  }
}

GET /test_index/_search 
{
  "size": 0, 
  "aggs": {
    "group_by_test_field": {
      "terms": {
        "field": "test_field"
      }
    }
  }
}
```

#### 使用内置field不分词，对string field进行聚合



```bash
GET /test_index/_search 
{
  "size": 0,
  "aggs": {
    "group_by_test_field": {
      "terms": {
        "field": "test_field.keyword"
      }
    }
  }
}
```

如果对不分词的field执行聚合操作，直接就可以执行，不需要设置fieldata=true

#### 分词field+fielddata的工作原理

doc value --> 不分词的所有field，可以执行聚合操作 --> 如果某个field不分词，那么在创建索引时（index-time）就会自动生成doc value --> 针对这些不分词的field执行聚合操作的时候，自动就会用doc value来执行

分词field，是没有doc value的，在index-time，如果某个field是分词的，那么是不会给它建立doc value正排索引的，因为分词后，占用的空间过于大，所以默认是不支持分词field进行聚合的

分词field默认没有doc value，所以直接对分词field执行聚合操作，是会报错的

对于分词field，必须打开和使用fielddata，完全存在于纯内存中，结构和doc value类似；如果是ngram或者是大量term，那么必将占用大量的内存。

如果一定要对分词的field执行聚合，那么必须将fielddata=true，然后es就会在执行聚合操作的时候，现场将field对应的数据，建立一份fielddata正排索引，fielddata正排索引的结构跟doc value是类似的，
 但是只会将fielddata正排索引加载到内存中来，然后基于内存中的fielddata正排索引执行分词field的聚合操作

如果直接对分词field执行聚合，报错，才会让我们开启fielddata=true，告诉我们，会将fielddata uninverted index，正排索引，加载到内存，会耗费内存空间

为什么fielddata必须在内存？因为大家自己思考一下，分词的字符串，需要按照term进行聚合，需要执行更加复杂的算法和操作，如果基于磁盘和os cache，那么性能会很差

### 12.23 fielddata内存控制以及circuit breaker断路器

#### fielddata核心原理

fielddata加载到内存的过程是懒加载的，对一个分词 field执行聚合时，才会加载，而且是field-level加载的；

一个index的一个field，所有doc都会被加载，而不是少数doc；不是index-time创建，是query-time创建

#### fielddata内存限制

- indices.fielddata.cache.size: 20%，超出限制，清除内存已有fielddata数据
- fielddata占用的内存超出了这个比例的限制，那么就清除掉内存中已有的fielddata数据
- 默认无限制，限制内存使用，但是会导致频繁evict和reload，大量IO性能损耗，以及内存碎片和gc

#### 监控fielddata内存使用

```swift
GET /_stats/fielddata?fields=*
GET /_nodes/stats/indices/fielddata?fields=*
GET /_nodes/stats/indices/fielddata?level=indices&fields=*
```

#### circuit breaker

如果一次query load的feilddata超过总内存，就会发生内存溢出（OOM）

circuit breaker会估算query要加载的fielddata大小，如果超出总内存，就短路，query直接失败

```css
indices.breaker.fielddata.limit：fielddata的内存限制，默认60%
indices.breaker.request.limit：执行聚合的内存限制，默认40%
indices.breaker.total.limit：综合上面两个，限制在70%以内
```

#### fielddata filter的细粒度内存加载控制

```bash
POST /test_index/_mapping
{
  "properties": {
    "my_field": {
      "type": "text",
      "fielddata": { 
        "filter": {
          "frequency": { 
            "min": 0.01, 
            "min_segment_size": 500  
          }
        }
      }
    }
  }
}
```

min：仅仅加载至少在1%的doc中出现过的term对应的fielddata

比如说某个值，hello，总共有1000个doc，hello必须在10个doc中出现，那么这个hello对应的fielddata才会加载到内存中来

min_segment_size：少于500 doc的segment不加载fielddata

加载fielddata的时候，也是按照segment去进行加载的，某个segment里面的doc数量少于500个，那么这个segment的fielddata就不加载

一般不会去设置它，知道就好

#### fielddata预加载机制以及序号标记预加载

如果真的要对分词的field执行聚合，那么每次都在query-time现场生产fielddata并加载到内存中来，速度可能会比较慢

我们是不是可以预先生成加载fielddata到内存中来？

#### fielddata预加载

```bash
POST /test_index/_mapping
{
  "properties": {
    "test_field": {
      "type": "string",
      "fielddata": {
        "loading" : "eager" 
      }
    }
  }
}
```

query-time的fielddata生成和加载到内存，变为index-time，建立倒排索引的时候，会同步生成fielddata并且加载到内存中来，
 这样的话，对分词field的聚合性能当然会大幅度增强

#### 序号标记预加载

global ordinal原理解释

```undefined
doc1: status1
doc2: status2
doc3: status2
doc4: status1
```

有很多重复值的情况，会进行global ordinal标记，类似下面

```rust
status1 --> 0
status2 --> 1
```

这样doc中可以这样存储

```undefined
doc1: 0
doc2: 1
doc3: 1
doc4: 0
```

建立的fielddata也会是这个样子的，这样的好处就是减少重复字符串的出现的次数，减少内存的消耗

```bash
POST /test_index/_mapping
{
  "properties": {
    "test_field": {
      "type": "string",
      "fielddata": {
        "loading" : "eager_global_ordinals" 
      }
    }
  }
}
```

### 12.24 海量bucket优化机制：从深度优先到广度优先

当buckets数量特别多的时候，深度优先和广度优先的原理

## 十三、数据建模实战

### 13.1 关系型与document类型数据模型对比

关系型数据库的数据模型：三范式 --> 将每个数据实体拆分为一个独立的数据表，同时使用主外键关联关系将多个数据表关联起来 --> 确保没有任何冗余的数据；

es的数据模型：类似于面向对象的数据模型，将所有由关联关系的数据，放在一个doc json类型数据中，整个数据的关系，还有完整的数据，都放在了一起。

### 13.2 通过应用层join实现用户与博客的关联

在构造数据模型的时候，还是将有关联关系的数据，然后分割为不同的实体，类似于关系型数据库中的模型

用户信息：

```bash
PUT /website-users/1 
{
  "name":     "小鱼儿",
  "email":    "xiaoyuer@sina.com",
  "birthday":      "1980-01-01"
}
```

用户发布的博客

```bash
PUT /website-blogs/1
{
  "title":    "我的第一篇博客",
  "content":     "这是我的第一篇博客，开通啦！！！"
  "userId":     1 
}
```

在进行查询时就属于应用层的join，在应用层先查出一份数据（查用户信息），然后再查出一份数据（查询博客信息），进行关联

优点和缺点

- 优点：数据不冗余，维护方便
- 缺点：应用层join，如果关联数据过多，导致查询过大，性能很差

### 13.3 通过数据冗余实现用户与博客的关联

```bash
PUT /website-users/1
{
  "name":     "小鱼儿",
  "email":    "xiaoyuer@sina.com",
  "birthday":      "1980-01-01"
}
```

这里面冗余用户名字段

```bash
PUT /website-blogs/_doc/1
{
  "title": "小鱼儿的第一篇博客",
  "content": "大家好，我是小鱼儿。。。",
  "userInfo": {
    "userId": 1,
    "username": "小鱼儿"
  }
}
```

冗余数据，就是将可能会进行搜索的条件和要搜索的数据，放在一个doc中

优点和缺点

- 优点：性能高，不需要执行两次搜索
- 缺点：数据冗余，维护成本高；比如某个字段更新后，需要更新相关的doc

### 13.4 对每个用户发表的博客进行分组

添加测试数据：

```bash
POST /website_users/_doc/3
{
  "name": "黄药师",
  "email": "huangyaoshi@sina.com",
  "birthday": "1970-10-24"
}

PUT /website_blogs/_doc/3
{
  "title": "我是黄药师",
  "content": "我是黄药师啊，各位同学们！！！",
  "userInfo": {
    "userId": 1,
    "userName": "黄药师"
  }
}

PUT /website_users/_doc/2
{
  "name": "花无缺",
  "email": "huawuque@sina.com",
  "birthday": "1980-02-02"
}

PUT /website_blogs/_doc/4
{
  "title": "花无缺的身世揭秘",
  "content": "大家好，我是花无缺，所以我的身世是。。。",
  "userInfo": {
    "userId": 2,
    "userName": "花无缺"
  }
}
```

对每个用户发表的博客进行分组

```bash
GET /website_blogs/_search 
{
  "size": 0, 
  "aggs": {
    "group_by_username": {
      "terms": {
        "field": "userInfo.userName.keyword"
      },
      "aggs": {
        "top_blogs": {
          "top_hits": {
            "_source": {
              "includes": "title"
            }, 
            "size": 5
          }
        }
      }
    }
  }
}
```

### 13.5 对文件系统进行数据建模以及文件搜索实战

数据建模，对类似文件系统这种的有多层级关系的数据进行建模

#### 文件系统数据构造

```bash
PUT /fs
{
  "settings": {
    "analysis": {
      "analyzer": {
        "paths": { 
          "tokenizer": "path_hierarchy"
        }
      }
    }
  }
}
```

path_hierarchy示例说明：当文件路径为`/a/b/c/d` 执行path_hierarchy建立如下的分词 `/a/b/c/d`, `/a/b/c`, `/a/b`, `/a`

```bash
PUT /fs/_mapping
{
  "properties": {
    "name": { 
      "type":  "keyword"
    },
    "path": { 
      "type":  "keyword",
      "fields": {
        "tree": { 
          "type":     "text",
          "analyzer": "paths"
        }
      }
    }
  }
}
```

添加数据

```bash
PUT /fs/_doc/1
{
  "name":     "README.txt", 
  "path":     "/workspace/projects/helloworld", 
  "contents": "这是我的第一个elasticsearch程序"
}
```

#### 对文件系统执行搜索

文件搜索需求：查找一份，内容包括elasticsearch，在/workspace/projects/hellworld这个目录下的文件

```bash
GET /fs/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "contents": "elasticsearch"
          }
        },
        {
          "constant_score": {
            "filter": {
              "term": {
                "path": "/workspace/projects/helloworld"
              }
            }
          }
        }
      ]
    }
  }
}
```

搜索需求2：搜索/workspace目录下，内容包含elasticsearch的所有的文件

```bash
GET /fs/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "contents": "elasticsearch"
          }
        },
        {
          "constant_score": {
            "filter": {
              "term": {
                "path.tree": "/workspace"
              }
            }
          }
        }
      ]
    }
  }
}
```

### 13.6 基于全局锁实现悲观锁并发控制

如果多个线程，都过来要给/workspace/projects/helloworld下的README.txt修改文件名，需要处理出现多线程的并发安全问题；

#### 全局锁的上锁

```csharp
PUT /fs/_doc/global/_create
{}
```

- fs: 你要上锁的那个index
- _doc: 就是你指定的一个对这个index上全局锁的一个type
- global: 就是你上的全局锁对应的这个doc的id
- _create：强制必须是创建，如果/fs/lock/global这个doc已经存在，那么创建失败，报错

删除锁

```csharp
DELETE /fs/_doc/global
```

这个其实就是插入了一条带ID的数据，操作完了再删除，这样其他的就可以继续操作（如果程序挂掉了,没有来得及删除锁咋整??）

### 13.7 基于document锁实现悲观锁并发控制

通过脚本来加锁，锁具体某个ID的文档

```bash
POST /fs/_doc/1/_update
{
  "upsert": { "process_id": 123 },
  "script": "if ( ctx._source.process_id != process_id ) { assert false }; ctx.op = 'noop';"
  "params": {
    "process_id": 123
  }
}
```

### 13.8 基于共享锁和排他锁实现悲观锁并发控制

共享锁和排他锁的说明（相当于读写分离）：

- 共享锁：这份数据是共享的，然后多个线程过来，都可以获取同一个数据的共享锁，然后对这个数据执行读操作
- 排他锁：是排他的操作，只能一个线程获取排他锁，然后执行增删改操作

### 13.9 基于nested object实现博客与评论嵌套关系

#### 做一个实验，引出来为什么需要nested object

```bash
PUT /website/_doc/6
{
  "title": "花无缺发表的一篇帖子",
  "content":  "我是花无缺，大家要不要考虑一下投资房产和买股票的事情啊。。。",
  "tags":  [ "投资", "理财" ],
  "comments": [ 
    {
      "name":    "小鱼儿",
      "comment": "什么股票啊？推荐一下呗",
      "age":     28,
      "stars":   4,
      "date":    "2016-09-01"
    },
    {
      "name":    "黄药师",
      "comment": "我喜欢投资房产，风，险大收益也大",
      "age":     31,
      "stars":   5,
      "date":    "2016-10-22"
    }
  ]
}
```

被年龄是28岁的黄药师评论过的博客，搜索

```bash
GET /website/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "comments.name": "黄药师" }},
        { "match": { "comments.age":  28      }} 
      ]
    }
  }
}
```

这样的查询结果不是我们期望的

object类型底层数据结构，会将一个json数组中的数据，进行扁平化；类似：

```json
{
  "title":            [ "花无缺", "发表", "一篇", "帖子" ],
  "content":             [ "我", "是", "花无缺", "大家", "要不要", "考虑", "一下", "投资", "房产", "买", "股票", "事情" ],
  "tags":             [ "投资", "理财" ],
  "comments.name":    [ "小鱼儿", "黄药师" ],
  "comments.comment": [ "什么", "股票", "推荐", "我", "喜欢", "投资", "房产", "风险", "收益", "大" ],
  "comments.age":     [ 28, 31 ],
  "comments.stars":   [ 4, 5 ],
  "comments.date":    [ 2016-09-01, 2016-10-22 ]
}
```

#### 引入nested object类型，来解决object类型底层数据结构导致的问题

修改mapping，将comments的类型从object设置为nested

```bash
PUT /website
{
  "mappings": {
      "properties": {
        "comments": {
          "type": "nested", 
          "properties": {
            "name":    { "type": "text"  },
            "comment": { "type": "text"  },
            "age":     { "type": "short"   },
            "stars":   { "type": "short"   },
            "date":    { "type": "date"    }
          }
        }
      }
    }
}
```

执行查询

```bash
GET /website/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "花无缺"
          }
        },
        {
          "nested": {
            "path": "comments",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "黄药师"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 28
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

### 13.10 对嵌套的博客评论数据进行聚合分析

聚合数据分析的需求1：按照评论日期进行bucket划分，然后拿到每个月的评论的评分的平均值

```bash
GET /website/_search 
{
  "size": 0, 
  "aggs": {
    "comments_path": {
      "nested": {
        "path": "comments"
      }, 
      "aggs": {
        "group_by_comments_date": {
          "date_histogram": {
            "field": "comments.date",
            "calendar_interval": "month",
            "format": "yyyy-MM"
          },
          "aggs": {
            "avg_stars": {
              "avg": {
                "field": "comments.stars"
              }
            }
          }
        }
      }
    }
  }
}
```

查询示例2

```bash
GET /website/_search 
{
  "size": 0,
  "aggs": {
    "comments_path": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "group_by_comments_age": {
          "histogram": {
            "field": "comments.age",
            "interval": 10
          },
          "aggs": {
            "reverse_path": {
              "reverse_nested": {}, 
              "aggs": {
                "group_by_tags": {
                  "terms": {
                    "field": "tags.keyword"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## 十四、高级操作（使用较少）

### 14.1 基于term vector深入探查数据的情

```bash
GET /twitter/tweet/1/_termvectors
GET /twitter/tweet/1/_termvectors?fields=text

GET /my_index/my_type/1/_termvectors
{
  "fields" : ["fullname"],
  "offsets" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}
```

### 14.2 深入剖析搜索结果的highlight高亮显示

#### 简单示例

```bash
GET /blog_website/_search 
{
  "query": {
    "match": {
      "title": "博客"
    }
  },
  "highlight": {
    "fields": {
      "title": {}
    }
  }
}
```

<em></em>表现，会变成红色，所以说你的指定的field中，如果包含了那个搜索词的话，就会在那个field的文本中，对搜索词进行红色的高亮显示

```bash
GET /blog_website/blogs/_search 
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": "博客"
          }
        },
        {
          "match": {
            "content": "博客"
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "title": {},
      "content": {}
    }
  }
}
```

> highlight中的field，必须跟query中的field一一对齐的

#### 三种highlight介绍

plain highlight，lucene highlight，默认

posting highlight，index_options=offsets

- （1）性能比plain highlight要高，因为不需要重新对高亮文本进行分词
- （2）对磁盘的消耗更少
- （3）将文本切割为句子，并且对句子进行高亮，效果更好

```bash
GET /blog_website/_search 
{
  "query": {
    "match": {
      "content": "博客"
    }
  },
  "highlight": {
    "fields": {
      "content": {}
    }
  }
}
```

其实可以根据你的实际情况去考虑，一般情况下，用plain highlight也就足够了，不需要做其他额外的设置；
 如果对高亮的性能要求很高，可以尝试启用posting highlight；
 如果field的值特别大，超过了1M，那么可以用fast vector highlight

#### 设置高亮html标签，默认是<em>标签

```bash
GET /blog_website/_search 
{
  "query": {
    "match": {
      "content": "博客"
    }
  },
  "highlight": {
    "pre_tags": ["<tag1>"],
    "post_tags": ["</tag2>"], 
    "fields": {
      "content": {
        "type": "plain"
      }
    }
  }
}
```

#### 高亮片段fragment的设置

```bash
GET /blog_website/_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "content" : {"fragment_size" : 150, "number_of_fragments" : 3, "no_match_size": 150 }
        }
    }
}
```

fragment_size: 你一个Field的值，比如有长度是1万，但是你不可能在页面上显示这么长；设置要显示出来的fragment文本判断的长度，默认是100；

number_of_fragments：你可能你的高亮的fragment文本片段有多个片段，你可以指定就显示几个片段

### 14.3 使用search template将搜索模板化

搜索模板，search template，高级功能，就可以将我们的一些搜索进行模板化，然后的话，每次执行这个搜索，就直接调用模板，给传入一些参数就可以了

#### 基础示例

```cpp
GET /website_blogs/_search/template
{
  "source": {
    "query": {
      "match": {
        "{{field}}": "{{value}}"
      }
    }
  },
  "params": {
    "field": "title",
    "value": "黄药师"
  }
}
```

这个部分可以改为脚本文件，替换为"file":"search_by_title"

```bash
    "query": {
      "match": {
        "{{field}}": "{{value}}"
      }
    }
```

#### 使用josn串

```cpp
GET /website_blogs/_search/template
{
  "source": "{\"query\": {\"match\": {{#toJson}}matchCondition{{/toJson}}}}",
  "params": {
    "matchCondition": {
      "title": "黄药师"
    }
  }
}
```

#### 使用join

```cpp
GET /website_blogs/_search/template
{
  "source": {
    "query": {
      "match": {
        "title": "{{#join delimiter=' '}}titles{{/join delimiter=' '}}"
      }
    }
  },
  "params": {
    "titles": ["黄药师", "花无缺"]
  }
}
```

类比：

```bash
GET /website_blogs/_search/
{
  "query": { 
    "match" : { 
      "title" : "黄药师 花无缺" 
    } 
  }
}
```

#### conditional

es的config/scripts目录下，预先保存这个复杂的模板，后缀名是.mustache，文件名是conditonal

内容如下：

```bash
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "{{text}}" 
        }
      },
      "filter": {
        {{#line_no}} 
          "range": {
            "line_no": {
              {{#start}} 
                "gte": "{{start}}" 
                {{#end}},{{/end}} 
              {{/start}} 
              {{#end}} 
                "lte": "{{end}}" 
              {{/end}} 
            }
          }
        {{/line_no}} 
      }
    }
  }
}
```

查询语句

```cpp
GET /website_blogs/_search/template
{
  "file": "conditional",
  "params": {
    "text": "博客",
    "line_no": true,
    "start": 1,
    "end": 10
  }
}
```

#### 保存search template

config/scripts，.mustache

提供一个思路

比如一般在大型的团队中，可能不同的人，都会想要执行一些类似的搜索操作；
 这个时候，有一些负责底层运维的一些同学，就可以基于search template，封装一些模板出来，然后是放在各个es进程的scripts目录下的；
 其他的团队，其实就不用各个团队自己反复手写复杂的通用的查询语句了，直接调用某个搜索模板，传入一些参数就好了

### 14.4 基于completion suggest实现搜索提示

suggest，completion suggest，自动完成，搜索推荐，搜索提示 --> 自动完成，auto completion

比如我们在百度，搜索，你现在搜索“大话西游” --> 百度，自动给你提示，“大话西游电影”，“大话西游小说”， “大话西游手游”

不需要所有想要的输入文本都输入完，搜索引擎会自动提示你可能是你想要搜索的那个文本

#### 初始化数据

```bash
PUT /news_website
{
  "mappings": {
      "properties" : {
        "title" : {
          "type": "text",
          "analyzer": "ik_max_word",
          "fields": {
            "suggest" : {
              "type" : "completion",
              "analyzer": "ik_max_word"
            }
          }
        },
        "content": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
}
```

completion，es实现的时候，是非常高性能的，其构建不是倒排索引，也不是正拍索引，就是单独用于进行前缀搜索的一种特殊的数据结构，
 而且会全部放在内存中，所以auto completion进行的前缀搜索提示，性能是非常高的。

```bash
PUT /news_website/_doc/1
{
  "title": "大话西游电影",
  "content": "大话西游的电影时隔20年即将在2017年4月重映"
}
PUT /news_website/_doc/2
{
  "title": "大话西游小说",
  "content": "某知名网络小说作家已经完成了大话西游同名小说的出版"
}
PUT /news_website/_doc/3
{
  "title": "大话西游手游",
  "content": "网易游戏近日出品了大话西游经典IP的手游，正在火爆内测中"
}
```

#### 执行查询

```bash
GET /news_website/_search
{
  "suggest": {
    "my-suggest" : {
      "prefix" : "大话西游",
      "completion" : {
        "field" : "title.suggest"
      }
    }
  }
}
```

直接查询

```bash
GET /news_website/_search
{
  "query": {
    "match": {
      "content": "大话西游电影"
    }
  }
}
```

### 14.5 使用动态映射模板定制自己的映射策略

比如我们本来没有某个type，或者没有某个field，但是希望在插入数据的时候，es自动为我们做一个识别，动态映射出这个type的mapping，包括每个field的数据类型，一般用的动态映射，dynamic mapping

这里有个问题，如果我们对dynamic mapping有一些自己独特的需求，比如es默认的，如经过识别到一个数字，field: 10，默认是搞成这个field的数据类型是long，再比如说，如果我们弄了一个field : "10"，默认就是text，还会带一个keyword的内置field。我们没法改变。

但是我们现在就是希望动态映射的时候，根据我们的需求去映射，而不是让es自己按照默认的规则去玩儿

dyanmic mapping template，动态映射模板

我们自己预先定义一个模板，然后插入数据的时候，相关的field，如果能够根据我们预先定义的规则，匹配上某个我们预定义的模板，那么就会根据我们的模板来进行mapping，决定这个Field的数据类型

#### 根据类型匹配映射模板

动态映射模板，有两种方式，第一种，是根据新加入的field的默认的数据类型，来进行匹配，匹配上某个预定义的模板；
 第二种，是根据新加入的field的名字，去匹配预定义的名字，或者去匹配一个预定义的通配符，然后匹配上某个预定义的模板

##### 根据默认类型来

```bash
PUT my_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "integers": {
          "match_mapping_type": "long",
          "mapping": {
            "type": "integer"
          }
        }
      },
      {
        "strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "fields": {
              "raw": {
                "type": "keyword",
                "ignore_above": 500
              }
            }
          }
        }
      }
    ]
  }
}
```

##### 根据字段名配映射模板

```bash
PUT /my_index 
{
  "mappings": {
    "dynamic_templates": [
      {
        "string_as_integer": {
          "match_mapping_type": "string",
          "match": "long_*",
          "unmatch": "*_text",
          "mapping": {
            "type": "integer"
          }
        }
      }
    ]
  }
}
```

### 14.6 学习使用geo point地理位置数据类型

设置类型

```bash
PUT /hotel
{
  "mappings": {
    "properties": {
      "location":{
        "type": "geo_point"
      }
    }
  }
}
```

添加数据

```bash
PUT /hotel/_doc/1
{
  "name":"四季酒店",
  "location":{
    "lat":30.558456,
    "lon":104.073273
  }
}
```

> lat: 纬度，lon：经度

```bash
PUT /hotel/_doc/2
{
  "name":"成都威斯凯尔凯特酒店",
  "location":"30.5841,104.061939"
}

PUT /hotel/_doc/3
{
  "name":"北京天安门广场",
  "location":{
    "lat":39.909187,
    "lon":116.397451
  }
}
```

> 纬度在前，经度在后

#### 查询范围内的数据（左上角和右下角的点组成的矩形内的坐标）

```bash
GET /hotel/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left": {
          "lat": 40,
          "lon": 100
        },
        "bottom_right":{
           "lat": 30,
          "lon": 106
        }
      }
    }
  }
}       
```

#### 查询包含成都，且在指定区域的数据

```bash
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "成都"
          }
        }
      ],
      "filter": {
        "geo_bounding_box": {
          "location": {
            "top_left": {
              "lat": 40,
              "lon": 100
            },
            "bottom_right": {
              "lat": 30,
              "lon": 106
            }
          }
        }
      }
    }
  }
} 
```

#### 搜索多个点组成的多边型内



```bash
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {"match_all": {}}
      ],
      "filter": [
        {
        "geo_polygon": {
          "location": {
            "points": [
              {
                "lat": 40,
              "lon": 100
              },
              {
               "lat": 30,
              "lon": 106
              },
              {
               "lat": 35,
              "lon": 120
              }
            ]
          }
        }
        }
      ]
    }
  }
}    
```

#### 搜索指定坐标100km范围内的

```bash
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "100km",
       
            "location": {
              "lat": 30,
              "lon": 116
            }
          }
        }
      ]
    }
  }
} 
```

#### 统计距离100~300米内的酒店数

```bash
GET /hotel/_search
{
"size": 0, 
  "aggs": {
    "agg_by_distance_range": {
      "geo_distance": {
        "field": "location",
        "origin": {
          "lat": 30,
          "lon": 106
        },
        "unit": "mi", 
        "ranges": [
          {
            "from": 100,
            "to": 300
          }
        ]
      }
    }
  }
}
```

## 十五、熟练掌握ES Java API

### 15.1 集群自动探查以及汽车零售店案例背景

#### client集群自动探查

默认情况下，是根据我们手动指定的所有节点，依次轮询这些节点，来发送各种请求的，如下面的代码，我们可以手动为client指定多个节点

```cpp
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("192.168.6.1", 9200)
    , new HttpHost("192.168.6.2", 9200)));
```

但是问题是，如果我们有成百上千个节点呢？难道也要这样手动添加吗？

因此es client提供了一种集群节点自动探查的功能，打开这个自动探查机制以后，es client会根据我们手动指定的几个节点连接过去，
 然后通过集群状态自动获取当前集群中的所有data node，然后用这份完整的列表更新自己内部要发送请求的node list。
 默认每隔5秒钟，就会更新一次node list。

```cpp
    // 老版本的写法
    Settings settings = Settings.builder()
            .put("cluster.name", "docker-cluster")
            // 设置集群节点自动发现
            .put("client.transport.sniff", true)
            .build();
```

注意，es client是不会将Master node纳入node list的，因为要避免给master node发送搜索等请求。

这样的话，我们其实直接就指定几个master node，或者1个node就好了，client会自动去探查集群的所有节点，而且每隔5秒还会自动刷新。

### 15.2 基于upsert实现汽车最新价格的调整

建立mapper



```bash
PUT /car_shop
{
  "mappings": {
      "properties": {
        "brand": {
          "type": "text",
          "analyzer": "ik_max_word",
          "fields": {
            "raw": {
              "type": "keyword"
            }
          }
        },
        "name": {
          "type": "text",
          "analyzer": "ik_max_word",
          "fields": {
            "raw": {
              "type": "keyword"
            }
          }
        }
      }
    }
}
```

Java代码实现存在则更新否则添加

```csharp
IndexRequest indexRequest = new IndexRequest("car_shop");
        indexRequest.id("1");
        indexRequest.source(XContentFactory.jsonBuilder()
                .startObject()
                .field("brand", "宝马")
                .field("name", "宝马320")
                .field("price", 320000)
                .field("produce_date", "2020-01-01")
                .endObject());

UpdateRequest updateRequest = new UpdateRequest("car_shop", "1");
updateRequest.doc(XContentFactory.jsonBuilder()
        .startObject()
        .field("price", 320000)
        .endObject()).upsert(indexRequest);

UpdateResponse response = restHighLevelClient.update(updateRequest, RequestOptions.DEFAULT);
System.out.println(response.getResult());
```

### 15.3 基于mget实现多辆汽车的配置与价格对比

场景：一般我们都可以在一些汽车网站上，或者在混合销售多个品牌的汽车4S店的内部，都可以在系统里调出来多个汽车的信息，放在网页上，进行对比

mget：一次性将多个document的数据查询出来，放在一起显示。



```bash
PUT /car_shop/_doc/2
{
    "brand": "奔驰",
    "name": "奔驰C200",
    "price": 350000,
    "produce_date": "2020-01-05"
}
```

Java代码：



```csharp
MultiGetRequest multiGetRequest = new MultiGetRequest();
multiGetRequest.add("car_shop", "1");
multiGetRequest.add("car_shop", "2");

MultiGetResponse multiGetResponse = restHighLevelClient.mget(multiGetRequest, RequestOptions.DEFAULT);
MultiGetItemResponse[] responses = multiGetResponse.getResponses();
for(MultiGetItemResponse response:responses){
    System.out.println(response.getResponse().getSourceAsMap());
}
```

### 15.4 基于bulk实现多4S店销售数据批量上传

业务场景：有一个汽车销售公司，拥有很多家4S店，这些4S店的数据，都会在一段时间内陆续传递过来，汽车的销售数据，
 现在希望能够在内存中缓存比如1000条销售数据，然后一次性批量上传到es中去。

Java代码：



```csharp
BulkRequest bulkRequest = new BulkRequest();

// 添加数据
JSONObject car = new JSONObject();
car.put("brand", "奔驰");
car.put("name", "奔驰C200");
car.put("price", 350000);
car.put("produce_date", "2020-01-05");
car.put("sale_price", 360000);
car.put("sale_date", "2020-02-03");
bulkRequest.add(new IndexRequest("car_sales").id("3").source(car.toJSONString(), XContentType.JSON));

// 更新数据
bulkRequest.add(new UpdateRequest("car_shop", "2").doc(jsonBuilder()
        .startObject()
        .field("sale_price", "290000")
        .endObject()));

// 删除数据
bulkRequest.add(new DeleteRequest("car_shop").id("1"));

BulkResponse bulk = restHighLevelClient.bulk(bulkRequest, RequestOptions.DEFAULT);
System.out.println(bulk.hasFailures() +" " +bulk.buildFailureMessage());
```

### 15.5 基于scroll实现月度销售数据批量下载

当需要从es中下载大批量的数据时，比如说做业务报表时需要将数据导出到Excel中，如果数据有几十万甚至是上百万条数据，此时可以使用scroll对大量的数据批量的获取和处理



```csharp
// 创建查询请求，设置index
SearchRequest searchRequest = new SearchRequest("car_shop");
// 设定滚动时间间隔,60秒,不是处理查询结果的所有文档的所需时间
// 游标查询的过期时间会在每次做查询的时候刷新，所以这个时间只需要足够处理当前批的结果就可以了
searchRequest.scroll(TimeValue.timeValueMillis(60000));

// 构建查询条件
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
searchSourceBuilder.query(QueryBuilders.matchQuery("brand", "奔驰"));
// 每个批次实际返回的数量
searchSourceBuilder.size(2);
searchRequest.source(searchSourceBuilder);

SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);

// 获取第一页的
String scrollId = searchResponse.getScrollId();
SearchHit[] searchHits = searchResponse.getHits().getHits();

int page = 1;
//遍历搜索命中的数据，直到没有数据
while (searchHits != null && searchHits.length > 0) {
    System.out.println(String.format("--------第%s页-------", page++));
    for (SearchHit searchHit : searchHits) {
        System.out.println(searchHit.getSourceAsString());
    }
    System.out.println("=========================");

    SearchScrollRequest scrollRequest = new SearchScrollRequest(scrollId);
    scrollRequest.scroll(TimeValue.timeValueMillis(60000));
    try {
        searchResponse = restHighLevelClient.scroll(scrollRequest, RequestOptions.DEFAULT);
    } catch (IOException e) {
        e.printStackTrace();
    }

    scrollId = searchResponse.getScrollId();
    searchHits = searchResponse.getHits().getHits();
}

// 清除滚屏任务
ClearScrollRequest clearScrollRequest = new ClearScrollRequest();
// 也可以选择setScrollIds()将多个scrollId一起使用
clearScrollRequest.addScrollId(scrollId);
ClearScrollResponse clearScrollResponse = restHighLevelClient.clearScroll(clearScrollRequest,RequestOptions.DEFAULT);
System.out.println("succeeded:" + clearScrollResponse.isSucceeded());
```

> 所有数据获取完毕之后，需要手动清理掉 scroll_id 。
>  虽然es 会有自动清理机制，但是 scroll_id 的存在会耗费大量的资源来保存一份当前查询结果集映像，并且会占用文件描述符。所以用完之后要及时清理

### 15.6 基于search template实现按品牌分页查询模板



```csharp
Map<String, Object> params = new HashMap<>(1);
params.put("brand", "奔驰");

SearchTemplateRequest templateRequest = new SearchTemplateRequest();
templateRequest.setScript("{\n" +
        "  \"query\": {\n" +
        "    \"match\": {\n" +
        "      \"brand\": \"{{brand}}\" \n" +
        "    }\n" +
        "  }\n" +
        "}\n");
templateRequest.setScriptParams(params);
templateRequest.setScriptType(ScriptType.INLINE);
templateRequest.setRequest(new SearchRequest("car_shop"));

SearchTemplateResponse templateResponse = restHighLevelClient.searchTemplate(templateRequest, RequestOptions.DEFAULT);
SearchHit[] hits = templateResponse.getResponse().getHits().getHits();
if(null!=hits && hits.length!=0){
    for (SearchHit hit : hits) {
        System.out.println(hit.getSourceAsString());
    }
}else {
    System.out.println("无符合条件的数据");
}
```

### 15.7 对汽车品牌进行全文检索、精准查询和前缀搜索



```csharp
@Test
public void fullSearch() throws IOException {
    
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.matchQuery("brand", "奔驰"));
    search(searchSourceBuilder);
    System.out.println("-----------------------------");

    searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.multiMatchQuery("宝马", "brand", "name"));
    search(searchSourceBuilder);
    System.out.println("-----------------------------");

    searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.prefixQuery("name", "奔"));
    search(searchSourceBuilder);
    System.out.println("-----------------------------");

}

private void search(SearchSourceBuilder searchSourceBuilder) throws IOException {
    SearchRequest searchRequest = new SearchRequest("car_shop");
    searchRequest.source(searchSourceBuilder);
    SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
    SearchHit[] searchHits = searchResponse.getHits().getHits();
    if(searchHits!=null && searchHits.length!=0){
        for (SearchHit searchHit : searchHits) {
            System.out.println(searchHit.getSourceAsString());
        }
    }
}
```

### 15.8 对汽车品牌进行多种的条件组合搜索



```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.boolQuery()
            .must(QueryBuilders.matchQuery("brand", "奔驰"))
            .mustNot(QueryBuilders.termQuery("name.raw", "奔驰C203"))
            .should(QueryBuilders.termQuery("produce_date", "2020-01-02"))
            .filter(QueryBuilders.rangeQuery("price").gte("280000").lt("500000"))
    );
    
SearchRequest searchRequest = new SearchRequest("car_shop");
searchRequest.source(searchSourceBuilder);
SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
SearchHit[] searchHits = searchResponse.getHits().getHits();
if(searchHits!=null && searchHits.length!=0){
 for (SearchHit searchHit : searchHits) {
     System.out.println(searchHit.getSourceAsString());
 }
}
```

### 基于地理位置对周围汽车4S店进行搜索

需要将字段类型设置坐标类型



```bash
POST /car_shop/_mapping
{
  "properties": {
      "pin": {
          "properties": {
              "location": {
                  "type": "geo_point"
              }
          }
      }
  }
}
```

添加数据



```bash
PUT /car_shop/_doc/5
{
    "name": "上海至全宝马4S店",
    "pin" : {
        "location" : {
            "lat" : 40.12,
            "lon" : -71.34
        }
    }
}
```

#### 搜索两个坐标点组成的一个区域

```java
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
searchSourceBuilder.query(QueryBuilders.geoBoundingBoxQuery("pin.location")
        .setCorners(40.73, -74.1, 40.01, -71.12));
```

#### 指定一个区域，由三个坐标点，组成，比如上海大厦，东方明珠塔，上海火车站

```java
searchSourceBuilder = new SearchSourceBuilder();
List<GeoPoint> points = new ArrayList<>();
points.add(new GeoPoint(40.73, -74.1));
points.add(new GeoPoint(40.01, -71.12));
points.add(new GeoPoint(50.56, -90.58));
searchSourceBuilder.query(QueryBuilders.geoPolygonQuery("pin.location", points));
```

#### 搜索距离当前位置在200公里内的4s店

```java
searchSourceBuilder = new SearchSourceBuilder();
searchSourceBuilder.query(QueryBuilders.geoDistanceQuery("pin.location")
        .point(40, -70).distance(200, DistanceUnit.KILOMETERS));    
```

