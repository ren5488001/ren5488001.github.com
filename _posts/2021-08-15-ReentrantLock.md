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
## ReenTrantLock

Doug Lea老爷子实现的基于CAS 的**可重入、独占试** 锁，与synchronized具有基本相同的语义和行为，但使用起来更加灵活，具备可打断、轮训、超时等特有功能。

# 功能要点

eureka分为server端和client端

## Sync锁

ReenTrantLock中Sync有FairSync公平锁和NonFairSync非公平锁两个实现。

在调用构造方法时可以声明是否调用公平锁。

```java
	#声明调用
·	ReentrantLock lock = new ReentrantLock(true);
	#构造源码
	public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

<img src="C:\Users\renkewei\Desktop\FairSync.png" alt="FairSync" style="zoom:80%;" />

以下按照各个方法功能的调用说明ReenTrantLock的实现

## lock()

```java
        public void lock() {
            //此处分别调用的公平锁或非公平锁的lock()
            sync.lock();
        }

        //公平锁FairSync的实现，直接去acquire尝试获取
        final void lock() {
            acquire(1);
        }
        //AQS设定的所有获得锁的公有方法
        public final void acquire(int arg) {
            /**
             * 此处分步骤解析
             * 1.tryAcquire尝试获取锁，如果成功返回true则直接结束
             * 2.tryAcquire失败返回false，执行addWaiter(AbstractQueuedSynchronizer.Node.EXCLUSIVE)为当前线程创建Node节点并进行排队
             * 3.调用acquireQueued去获取锁
             */
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }
		
		// 1.tryAcquire尝试获取锁
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取当下lock的state
            int c = getState();
            //state=0 说明没有线程占用当前lock
            if (c == 0) {
                // 此处步骤分解
                // 1. hasQueuedPredecessors 查询当前节点是不是等待时间最长的节点，此处也体现了公平锁，等待时间最长的最先获取锁
                // 2. cas变更state值
                // 3. state变更成功则认为获取lock成功，执行setExclusiveOwnerThread设定当前线程为lock的独占线程，变更失败直接返回false
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //state!=0 说明已经有线程占用了lock，且当前线程就是lock的独占线程时，对state进行+1cas重写，体现了ReenTrantLock的可重入性。state此处累加，相应的unlock()时也需要依次解锁-1
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                // 此处场景，后续补充说明
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
		
		// 为当前线程创建Node节点并进行排队
		private Node addWaiter(Node mode) {
            Node node = new Node(Thread.currentThread(), mode);
            Node pred = tail;
            // 查询尾部节点tail存在则设定node的前置节点为tail，并cas将tail变更为node
            if (pred != null) {
                node.prev = pred;
                if (compareAndSetTail(pred, node)) {
                    pred.next = node;
                    return node;
                }
            }
            // 没有tail节点时，说明队列尚未初始化
            // 查看enq(node) 可知初始化队列是将头部head设了一个new Node(),没有实际线程驻存，当前线程节点是保存在第二位，head的next节点，此处也印证了acquireQueued中当前线程节点node的prev是head时就进行tryAcquire，head没有进行竞争，head本身就是空的
            enq(node);
            return node;
        }
		
		// 已死循环方式，去竞争
        final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                boolean interrupted = false;
                for (;;) {
                    final Node p = node.predecessor();
                    // 当前置节点是head时，tryAcquire 细节查看上面注释
                    if (p == head && tryAcquire(arg)) {
                        setHead(node);
                        p.next = null; // help GC
                        failed = false;
                        return interrupted;
                    }
                    // 1. shouldParkAfterFailedAcquire 前置节点检查waitStatus是不是SIGNAL
                    // 2. parkAndCheckInterrupt
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }
        //非公平锁NonfairSync的实现
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

**注册** client端启动时会将网络ip、端口号等信息主动注册，server将注册信息保存。

**获取服务注册列表**  获取server维护的服务列表，方便内部调用所依赖的其他client的接口，定时更新，然后缓存在本地。

**心跳** 通过心跳机制确认生存，若长期获取不到心跳，则任务该实例死亡。

**调用** 解析列表，获取真实接口连接，调用接口。

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
