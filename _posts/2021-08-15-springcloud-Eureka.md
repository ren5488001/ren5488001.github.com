---
layout: post
title:  "springcloud组件之Eureka"
date:   2021-8-15 9:40:00 +0800
categories: springcloud组件
tag: springcloud组件
---

* content
{:toc}
# 概念

## 注册中心

实现服务治理，即管理所有的服务信息和状态。

## Eureka

是一个RESTful风格的服务，是一个用于服务发现和注册的基础组件，是搭建Spring Cloud微服务的前提之一，它屏蔽了Server和client的交互细节，使得开发者将精力放到业务上。

就是所有分布式部署的server工程，可以将工程信息注册到这里，方便各个模块之间内部调用。

## 其余注册中心方案

Nacos，Consul，Zookeeper等

# 内容

eureka分为server端和client端

## client端

**注册** client端启动时会将网络ip、端口号等信息主动注册，server将注册信息保存。

**获取服务注册列表**  获取server维护的服务列表，方便内部调用所依赖的其他client的接口，定时更新，然后缓存在本地。

**心跳** 通过心跳机制确认生存，若长期（30s一次，连续三次90s）获取不到心跳，则任务该实例死亡。

**调用** 解析列表，获取真实接口连接，调用接口。



同步时间时延，客户端的操作同步到所有eureka客户机可能需要2分钟

通讯机制：http协议下的rest请求

## server端

**服务注册列表** 维护client注册的服务信息。

**服务检查** 定期检查心跳，长期失去联系，则将client从服务列表中移除。

主动注册自身服务信息去server端，获取所依赖的server信息，方便直接调用。自己具备一定的负载均衡功能。

# 单体部署步骤

## server端

1. pom.xml

   ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
   ```

2. application.yaml

   server端配置如下

   ```yaml
   #注册中心
   eureka:
     client:
     #设置服务注册中心的URL
       service-url:
         defaultZone: http://root:root@localhost:7900/eureka/
         #是否将自己注册到Eureka Server,默认为true，由于当前就是server，故而设置成false，表明该服务不会向eureka注册自己的信息
       register-with-eureka: false
       #是否从eureka server获取注册信息，由于单节点，不需要同步其他节点数据，用false
       fetch-registry: false
   ```

3. 代码

   启动类添加注释

   ```java
   @EnableEurekaServer
   ```

## clent端

1. pom.xml

   ```xml
    <dependency>
    		<groupId>org.springframework.cloud</groupId>
    		<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
   ```

   

2. application.yaml配置文件

   ```yaml
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:7001/eureka/ # 指定eureka 服务端交互地址
    #主动向server端注册和获取服务列表功能都是默认开启的，不需要专门配置
   ```

3. 代码

   启动类添加注释

   ```java
   @EnableEurekaClient
   ```

4. 注册成功

   ```
   2021-08-15 20:04:42.176  INFO 18732 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_UNKNOWN/client-001 - registration status: 204
   ```

# 高可用部署

单个server存在风险，同时启动多个server并互相注册，server节点间会互相同步信息，保证服务列表的一致性。

眼下时间紧迫，后续补充。
