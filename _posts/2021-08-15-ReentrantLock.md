---
layout: post
title:  "JUC之ReenTrantLock"
date:   2021-8-15 9:40:00 +0800
categories: JUC
tag: JUC
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

