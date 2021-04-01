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
    - Apache Spark
    - Apache Hbase
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

### JanusGraph Vertex

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

### JanusGraph Edge

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

### JanusGraph源码数据案列分析

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

##### GraphSON Format
```
数据示例:
--------------------------------------------------------------------
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
--------------------------------------------------------------------
根据上面示例的结构,可以直观的分析出,像这样的一条JSON,即包含了Vertex节点数据[id, label, properties],又带上了其相关联Edge数据
```
> **注意:** 在JanusGraph中,property属性也是类似节点的存在,同样也有id,由于Vertex ID唯一;
所以在考虑使用GraphSON Format 批量导入数据时,应着重考虑id分配的问题,不过笔者建议,不要轻易使用该Format方式,容易出问题

##### Script Format
```
数据示例:
--------------------------------------------------------------------
1:person:marko:29 knows:2:0.5,knows:4:1.0,created:3:0.4
2:person:vadas:27
3:project:lop:java
4:person:josh:32 created:3:0.4,created:5:1.0
5:project:ripple:java
6:person:peter:35 created:3:0.2  
--------------------------------------------------------------------

Groovy脚本实例:
def parse(line) {
    def parts = line.split(/ /)
    def (id, label, name, x) = parts[0].split(/:/).toList()
    def v1 = graph.addVertex(T.id, id, T.label, label)
    if (name != null) v1.property('name', name) // first value is always the name
    if (x != null) {
        // second value depends on the vertex label; it's either
        // the age of a person or the language of a project
        if (label.equals('project')) v1.property('lang', x)
        else v1.property('age', Integer.valueOf(x))
    }
    if (parts.length == 2) {
        parts[1].split(/,/).grep { !it.isEmpty() }.each {
            def (eLabel, refId, weight) = it.split(/:/).toList()
            def v2 = graph.addVertex(T.id, refId)
            v1.addOutEdge(eLabel, v2, 'weight', Double.valueOf(weight))
        }
    }
    return v1
}
--------------------------------------------------------------------
示例数据以及执行脚本开发思路都很清洗,这里就不再过多赘述,实际开发可以根据业务进行调整
```
> **注意:** 在使用addOutEdge()/addInEdge方法创建edge时,尽量梳理清两个Vertex间Edge的direction,
建议使用单方向的Direction,避免读写数据时造成逻辑混乱

### JanusGraph Input/Output Formats使用方法
```
示例:
--------------------------------------------------------------------
--> $JANUSGRAPH_HOME/bin/gremlin.sh
> outputGraphConfig='conf/janusgraph-xxx.properties'
> readGraph = GraphFactory.open('conf/hadoop-graph/hadoop-xxx.properties')
> blvp = BulkLoaderVertexProgram.build().writeGraph(outputGraphConfig).create(readGraph)
> readGraph.compute(SparkGraphComputer).workers(2).program(blvp).submit().get()
```
> **注:** 上述涉及到一些目录配置,不清楚的可以先阅读 **[服务搭建篇](./2021-03-02-JanusGraph使用心得-服务搭建篇.md)**   

---

除上述bulk load方式外,通过Apache Spark分布式处理,效率上也还行,
虽然数据导入实际是通过JanusGraph连接Apache Hbase Client方式,若数据量没有达到几十亿个Vertex情况下,该方案暂时可行;
叙述到这,了解Hbase的小伙伴一定好奇为什么没用生成Hfile形式进行导入,而且这种方式更高效,
这里先卖个关子,详细情况可去阅读 **[Hbase存储篇](./2021-03-06-JanusGraph使用心得-Hbase存储篇.md)**

---
> [JanusGraph官网](https://docs.janusgraph.org/) | [TinkerPop Documentation](https://tinkerpop.apache.org/docs/3.4.6/reference/#order-step) | [Gremlin Practical doc](https://kelvinlawrence.net/book/Gremlin-Graph-Guide.html#exedge)