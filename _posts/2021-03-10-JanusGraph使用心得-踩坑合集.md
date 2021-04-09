---
layout:     post
title:      JanusGraph使用心得-踩坑合集
subtitle:   
date:       2021-03-12
author:     假正经
header-img: img/banner_janusgraph.png
catalog: true
tags:
    - JanusGraph
    - Apache Hbase
    - ElasticSearch
---

从事图数据开发,入坑已经一年多,在众多图数据库技术中,JanusGraph开源又免费的身份,使得有不少开发者青睐,但总体上讲,依旧有不少缺点

本篇文章会陆续更新下去

# 踩坑合集

- JanusGraph与Hbase,ES版本存在兼容问题，具体情况可浏览官网文档给的Version Compatibility
    ```
    目前生产环境使用版本如下:
        JanusGraph: 0.3.3    Apache Hbase: 1.3.6    ElasticSearch: 6.6.0
    ```
- 因机器部署的Spark环境中 jars目录使用的guava版本过低，导致客户端无法进行数据读写   
  `spark-2.1.1版本中使用的guava版本是guava-14.0.1.jar`
  

- JanusGraph 0.3.3安装包中的配置文件janusgraph-hbase-es.properties无法使用，原因是少了一个固定配置   
  `gremlin.graph=org.janusgraph.core.JanusGraphFactory`
  

- 客户端进行数据操作时，因事务未正常提交，导致操作无效
  

- 因数据重复操作，导致图数据出现重复
  

- 索引创建后，状态异常，未生效  
  ![graph index life cycle](/img/graph_inde_life_cycle.png)
  ```
    INSTALLED：索引已安装在系统中，但尚未在集群中的所有实例中注册
    REGISTERED：索引已在集群中的所有实例中注册，但尚未启用
    ENABLED：索引已启用并正在使用
    DISABLED：索引已禁用，不再使用
  ```

- 服务配置中，未更新ES的相关配置，导致服务启动后按默认副本数及分片数创建索引    
  `配置默认分片数为5，副本数为2`
  

- Hbase Region因end_key为空，导致JanusGraph在读取配置数据时抛出异常
  ```
  JanusGraph 配置项中有这么一个参数：storage.hbase.skip-schema-check,作用是跳过服务启动时访问Hbase table的表信息 
  注意：在初始运行JanusGraph时该参数默认为false,并在Hbase中根据源码配置的region数据及family创建相应的table
  ```
- 因未更新query.force-index配置，导致全图扫描
  ```html
  默认值为false,在未使用索引情况下，vertex or edge的查询都会进行全图搜索,
  建议服务启动时,应将该配置修改为 "true"
  ```
- 因JanusGraph中graph具有原子性，在重新创建graph时，需要将连接JanusGraph服务的系统进行重启
  

- 浏览过早期涉及JanusGraph的技术博客，发现其中会有一些问题

  1. 出现配置名称有误，可能是早期版本使用的配置名称，但建议修改配置时直接浏览JanusGraph官网 [[**JanusGraph Configuration Reference**]](https://docs.janusgraph.org/basics/configuration-reference/)
  

- JanusGraph Client开发所依赖jar包，官网似乎给的不全，这里罗列基于Spark读写JanusGraph所需要的jar包
    ```
        <dependency>
            <groupId>org.apache.tinkerpop</groupId>
            <artifactId>spark-gremlin</artifactId>
            <version>3.3.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-graphx_2.11</artifactId>
            <version>2.4.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5</version>
        </dependency>
        <dependency>
            <groupId>org.janusgraph</groupId>
            <artifactId>janusgraph-core</artifactId>
            <version>0.3.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.tinkerpop</groupId>
            <artifactId>gremlin-driver</artifactId>
            <version>3.4.1</version>
        </dependency>
        <dependency>
            <groupId>org.janusgraph</groupId>
            <artifactId>janusgraph-all</artifactId>
            <version>0.3.3</version>
        </dependency>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.38.Final</version>
        </dependency>
    ```
  - 若客户端使用jar包版本与部署服务版本不一致，将会获取不到数据

---
> [**JanusGraph Verson Compatibility**](https://docs.janusgraph.org/changelog/)