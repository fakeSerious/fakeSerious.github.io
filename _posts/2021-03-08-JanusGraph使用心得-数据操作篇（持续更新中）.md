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

本篇主要梳理了目前对数据进行数据读写操作使用的方法,也曾多次浏览JanusGraph相关网站文档,基于目前公司业务对JanusGraph的使用情况，暂不需要考虑数据读写性能的问题
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
> **注**: 属性<**bulkLoader.vertex.id**>必须创建,不然在导入新Vertex时会报错

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


---
>[JanusGraph官网](https://docs.janusgraph.org/) | [TinkerPop Documentation](https://tinkerpop.apache.org/docs/3.4.6/reference/#order-step) | [Gremlin Practical doc](https://kelvinlawrence.net/book/Gremlin-Graph-Guide.html#exedge)