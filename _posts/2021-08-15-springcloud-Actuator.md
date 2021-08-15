---
layout: post
title:  "springcloud组件之Actuator"
date:   2021-8-15 20:40:00 +0800
categories: springcloud组件Actuator
tag: springcloud组件Actuator
---

* content
{:toc}
# 概念

## Actuator

监视和管理工程各项指标的组件。

## 端点endpoints 

Actuator提供的端点，主要作用有查看指标情况、查看配置信息、操作控制工程。

# 部署步骤

1. pom.xml

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   ```

2. application.yaml

   actuator默认开启了health和info端点，其余端点需要专门开启

   server端配置如下

   ```yaml
   #actuator
   #开启所有端点，现网要根据实际情况开放
   management:
     endpoints:
       web:
         exposure:
           include: *
   ```

# endpoint端点

| HTTP方法 | 路径            | 描述                       | 鉴权  |
| :------: | :-------------- | :------------------------- | ----- |
|   GET    | /autoconfig     | 查看自动配置的使用情况     | true  |
|   GET    | /configprops    | 查看配置属性，包括默认配置 | true  |
|   GET    | /beans          | 查看bean及其关系列表       | true  |
|   GET    | /dump           | 打印线程栈                 | true  |
|   GET    | /env            | 查看所有环境变量           | true  |
|   GET    | /env/{name}     | 查看具体变量值             | true  |
|   GET    | /health         | 查看应用健康指标           | false |
|   GET    | /info           | 查看应用信息               | false |
|   GET    | /mappings       | 查看所有url映射            | true  |
|   GET    | /metrics        | 查看应用基本指标           | true  |
|   GET    | /metrics/{name} | 查看具体指标               | true  |
|   POST   | /shutdown       | 关闭应用                   | true  |
|   GET    | /trace          | 查看基本追踪信息           | true  |



