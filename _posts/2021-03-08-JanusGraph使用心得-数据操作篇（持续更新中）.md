---
layout:     post
title:      JanusGraph使用心得-数据操作 (持续更新中)
subtitle:   
date:       2021-03-08
author:     假正经
header-img: img/banner_janusgraph.png
catalog: true
tags:
    - JanusGraph
    - Gremlin
    - Apache Groovy Programming Language
---

本篇主要梳理了目前对数据进行数据读写操作使用的方法,也曾多次浏览JanusGraph相关网站文档,基于目前公司业务对JanusGraph的使用情况,暂不需要考虑数据读写性能的问题
并且本篇涉及到Graph Traveral Step较少,后续也会陆续更新


# Java connect JanusGraph
```shell
方式1:
import org.janusgraph.core.JanusGraphFactory
val connect = JanusGraphFactory.build
      .set("storage.backend", "your.graph.storage.backend")
      .set("index.search.backend", "your.graph.index.search.backend")
      .set("index.search.hostname", "your.graph.index.search.hostname")
      .set("index.search.elasticsearch.create.ext.number_of_shards","your.graph.index.search.elasticsearch.create.ext.number_of_shards")
      .set("index.search.elasticsearch.create.ext.number_of_replicas","your.graph.index.search.elasticsearch.create.ext.number_of_replicas")
      .set("storage.hostname", "your.graph.hosts")
      .set("storage.hbase.table", "your.graph.storage.hbase.table")
      .set("query.force-index", "your.graph.query.force-index")
      .open

方式2:
import org.apache.commons.configuration.PropertiesConfiguration
import org.apache.tinkerpop.gremlin.driver.Cluster
val configuration = new PropertiesConfiguration
    configuration.setProperty("hosts", "graph.hosts")
    configuration.setProperty("port", "graph.port")
    configuration.setProperty("connectionPool.minSize", "graph.connectionPool.minSize")
    configuration.setProperty("connectionPool.maxSize", "graph.connectionPool.maxSize")
    configuration.setProperty("connectionPool.maxInProcessPerConnection","graph.connectionPool.maxInProcessPerConnection")
    configuration.setProperty("connectionPool.maxSimultaneousUsagePerConnection","graph.connectionPool.maxSimultaneousUsagePerConnection")
    configuration.setProperty("connectionPool.maxContentLength","graph.connectionPool.maxContentLength")
    configuration.setProperty("serializer.className", "org.apache.tinkerpop.gremlin.driver.ser.GryoMessageSerializerV3d0")
    configuration.setProperty("serializer.config.ioRegistries", "org.janusgraph.graphdb.tinkerpop.JanusGraphIoRegistry,org.janusgraph.graphdb.tinkerpop.JanusGraphIoRegistry")
Cluster.open(configuration)
```
---
# Create Graph Schema **By Groovy**
```
def defineGratefulDeadSchema(janusGraph) {
    m = janusGraph.openManagement()

    /**---- example: create vertex labels ----*/
    vertex_label_name = m.makeVertexLabel("vertex_label_name").make()
    ...
    
    /**---- example: create edge labels ----*/
    edge_label_name = m.makeEdgeLabel("edge_label_name").make()
    ...

    /**---- example: create vertex and edge properties ----*/
    blid = m.makePropertyKey("bulkLoader.vertex.id").dataType(String.class).cardinality(Cardinality.SINGLE).make()
    property_key = m.makePropertyKey("property_key").dataType(String.class).cardinality(Cardinality.SINGLE).make()
    ...

    /**---- example: create global indices -> composite index ----*/
    m.buildIndex("byBulkLoaderVertexId", Vertex.class).addKey(blid).buildCompositeIndex()
    
    /**---- example: create global indices -> mixed index ----*/
    m.buildIndex("index_name", Vertex.class).addKey("property_key_name", Mapping.STRING.asParameter()).indexOnly("vertex_label_name").buildMixedIndex("search")

    /**---- example: create vertex centric indices ----*/
    m.buildEdgeIndex("edge_label_name", "index_name", Direction.BOTH, Order.decr, weight)
    
    /**---- 提交事务 ----*/
    m.commit()
}

Gremlin 命令执行schema创建:
    > :load ./xxx-schema.groovy
    > t = JanusGraphFactory.open('conf/janusgraph-xxx.properties')
    > defineGratefulDeadSchema(t)
    > t.close()
```

> **注意:** 属性<**bulkLoader.vertex.id**>必须创建,不然在导入新Vertex时会报错

---
## JanusGraph Vertex

```shell
i.新增节点
    g.addV("vertex_label_name").property("property_key","property_value").next()
ii.查询
    val v1 = g.V().has("property_key","property_value").next()
iii.更新节点
    g.V(v1).property("property_key","after_value")
iiii.删除节点
    g.V(v1).drop().iterator()
```

> **注意:** 当删除节点时,若未加上iterator()方法,从不会自动遍历遍历的意义上讲,该步骤不会终止;因此,必须进行某种形式的迭代才能真正进行删除,建议参照上面的代码案例进行开发操作

---
## JanusGraph Edge

```shell
i.新增edge
    g.V(v2).as("v2").V(v1).addE("edge_label_name").property("property_key","property_value").from("v2").next()
ii.查询
    方式1:
        g.E("edge_id")
    方式2:
        import org.apache.tinkerpop.gremlin.process.traversal.dsl.graph.__.inV
        import org.apache.tinkerpop.gremlin.process.traversal.P
        g.V(v1).outE("edge_label_name").where(inV().has("vertex_property_key",P.eq("vertex_property_value")))
        g.V(v1).out("edge_label_name").has("edge_property_key","edge_property_value")
iii.更新edge
    g.E("edge_id").property("property_key","after_value")
iiii.删除edge
    g.E("edge_id").drop().iterator()
```

---
# Groovy脚本实现Vertex批量导入
```shell
i.批量导入生成Vertex
def parse(line) {
  def (vertex, outEdges, inEdges) = line.split(/\t/, 3)
  def (v1id, v1label, v1props) = vertex.split(/,/, 3)
  def v1 = graph.addVertex(T.id, v1id.toInteger(), T.label, v1label)
  switch (v1label) {
      case "song":
          def (name, songType, performances) = v1props.split(/,/)
          v1.property("name", name)
          ...
          v1.property("performances", performances.toInteger())
          break
      case "artist":
          v1.property("name", v1props)
          break
      default:
          throw new Exception("Unexpected vertex label: ${v1label}")
  }
  return v1
}
```

## JanusGraph源码数据案列分析

目前JanusGraph支持Input/Output Formats有Gryo,GraphSON,Script
**[Input/Output Formats doc](https://tinkerpop.apache.org/docs/3.4.6/reference/#_input_output_formats)**

```
以下是不同Format的数据类型:
    Gryo Format:        经过Kyro序列化后的数据
    GraphSON Format:    JSON
    Script Format:      txt文本
```
在工作中,为实现图数据快速导入的要求下,利用JanusGraph源生**bulk load**方案也是不错的选择    
由于Gryo Format的数据无法直观分析,下面主要介绍GraphSON Format 和Script Format

### GraphSON Format
```
{
  "id": 1,
  "label": "person",
  "outE": {
    "created": [ { "id": 9, "inV": 3, "properties": { "weight": 0.4 } } ],
    "knows": [
      { "id": 7, "inV": 2, "properties": { "weight": 0.5 } },
      ...
    ]
  },
  "inE": {
    ...
  },
  "properties": {
    "name": [ { "id": 0, "value": "marko" } ],
    "age": [ { "id": 1, "value": 29 } ]
  }
}
根据上面示例的结构,可以直观的分析出,像这样的一条JSON,即包含了Vertex节点数据[id, label, properties],又带上了其相关联Edge数据
```
> **注意:** 在JanusGraph中,property属性也是类似节点的存在,同样也有id,由于Vertex ID唯一
所以在考虑使用GraphSON Format 批量导入数据时,应着重考虑id分配的问题,不过笔者建议不要轻易使用该Format方式,容易出问题

### Script Format

---
> [JanusGraph官网](https://docs.janusgraph.org/) | [TinkerPop Documentation](https://tinkerpop.apache.org/docs/3.4.6/reference/#order-step) | [Gremlin Practical doc](https://kelvinlawrence.net/book/Gremlin-Graph-Guide.html#exedge)