---
layout:     post
title:      JanusGraph使用心得-服务搭建
subtitle:   
date:       2021-03-02
author:     假正经
header-img: img/banner_janusgraph.png
catalog: true
tags:
    - JanusGraph
    - Apache Hbase
    - ElasticSearch
---

# JanusGraph 服务搭建

### 目录&配置简述

- bin
  ```
  gremlin-server.sh
  gremlin.sh
  janusgraph.sh
  ```
- conf
  ```
  janusgraph-berkeleyje-es.properties   
  janusgraph-cassandra.properties   
  janusgraph-cql.properties   
  janusgraph-hbase-es.properties    
  janusgraph-hbase.properties
  ...   
  ```
  - gremlin-server    
    ```
    gremlin-server-berkeleyje-es.yaml   
    gremlin-server-berkeleyje.yaml    
    gremlin-server-configuration.yaml   
    gremlin-server.yaml   
    ...
    ```
  - cassandra
  - hadoop-graph
    ```
    实现JanusGraph支持的Import/Output Format批量操作的配置  
    hadoop-graphson.properties    
    hadoop-gryo.properties    
    hadoop-load.properties    
    hadoop-script.properties    
    read-cassandra-3.properties   
    read-cassandra.properties   
    read-hbase.properties   
    read-hbase-snapshot.properties
    ```
  - solr
- data
  <font size=2 color="#FF6651">包含groovy脚本文件,以及支持数据批量装载的文件类型(kryo,json,txt,xml等)</font>  
- elasticsearch
  <font size=2 color="#FF6651">自带兼容版本的ES服务包,JanusGraph各版本之间对ES,Hbase等兼容并不一致,在选择图后端存储/图索引存储时，应注意版本是否兼容</font>
- examples
- ext
- javadocs
  <font size=2 color="#FF6651">涉及JanusGraph相关类,接口等</font>
- lib
  <font size=2 color="#FF6651">依赖jar包存放路径</font>
- run
  <font size=2 color="#FF6651">服务运行时，默认会产生pid文件并将其保存在pid目录中，也可对shell执行文件进行修改，更新pid文件存在路径</font>
- scripts

---
### 服务搭建
```
使用版本：
JanusGraph 0.3.3    Apache Hbase 1.3.6    ElasticSearch 6.6.0
```
##### 单机部署
```
配置方案: ./conf/janusgraph-hbase-es.properties

部署步骤如下:
  1. 解压JabusGraph部署包
      > :  unzip janusgraph-0.3.3-hadoop2.zip
  2. 调整配置文件$JANUSGRAPH_HOME/conf/janusgraph-hbase-es.properties
      #配置后端存储-Apache Hbase
      storage.backend=hbase
      storage.batch-loading=true
      ids.block-size=20000000
      ids.renew-timeout=3600000
      storage.buffer-size=20480

      #Apache Hbase -> ZK_HOST
      storage.hostname=xxx.xx.xx.xx,xxx.xx.xx.xx,...
      storage.hbase.table=rads:h_enterprise_graph
      storage.hbase.ext.zookeeper.znode.parent=/hbase-secure
      storage.hbase.region-count=9
      storage.hbase.ext.hbase.rpc.timeout = 300000
      storage.hbase.ext.hbase.client.operation.timeout = 300000
      storage.hbase.ext.hbase.client.scanner.timeout.period = 300000
      #JanusGraph初始运行时该配置必须保持默认值false
      storage.hbase.skip-schema-check=true

      cache.db-cache = false
      cache.db-cache-clean-wait = 20
      cache.db-cache-time = 180000
      cache.db-cache-size = 0.5
      
      #配置混合索引-ElasticSearch
      index.search.backend=elasticsearch
      index.search.hostname=xxx.xx.xx.xx
      index.search.port=9200
      index.search.elasticsearch.create.ext.number_of_shards=1
      index.search.elasticsearch.create.ext.number_of_replicas=0
      
      #更多配置信息可以浏览JanusGraph官网
      
  3. 调整服务配置文件conf/gremlin-server/gremlin-server.yaml
      host: xxx.xx.xx.xx
      port: 8182
      scriptEvaluationTimeout: 60000
      channelizer: org.apache.tinkerpop.gremlin.server.channel.WebSocketChannelizer
      graphs: {graph: conf/janusgraph-hbase-es.properties}
      #其余配置可默认不变
  
  4. 将以下配置文件拷贝一份到目录conf/gremlin-server/中
      core-site.xml
      hbase-site.xml
      hdfs-site.xml
      
  5. 启动/关闭服务
      >: $JANUSGRAPH_HOME/bin/gremlin-server.sh start
      >: $JANUSGRAPH_HOME/bin/gremlin-server.sh stop
```

##### 集群部署
    部署流程细节与单机部署{1,2,4}步骤基本一致,仅需将步骤3中的host指定相应的host即可

### 常用命令
```
JanusGraph Gremlin Shell 启动: $JANUSGRAPH_HOME/bin/gremlin.sh

Opens a JanusGraph Database Instance: Graph=JanusGraphFactory.open('$JANUSGRAPH_HOME/conf/janusgraph-hbase-es.properties')
  
open the management system for this graph instance: mgmt = graph.openManagement()

查看Graph Schema: mgmt. printSchema()

生成可重复使用的GraphTraversalSource实例: g = graph.traversal()
```

> [JanusGraph官网](https://docs.janusgraph.org/)