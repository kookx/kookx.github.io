---
layout: post
title: ZooKeeper入门
category: Blog
tags: [分布式锁,ZooKeeper,微服务]
---

## 概述

ZooKeeper是一种为分布式应用所设计的高可用、高性能且一致的开源协调服务，它提供了一项基本服务：**分布式锁服务**。由于ZooKeeper的开源特性，后来我们的开发者在分布式锁的基础上，摸索了出了其他的使用方法：**配置维护、组服务、分布式消息队列**、**分布式通知/协调**等。

### ZooKeeper特性

- **简单**

Zookeeper的核心是一个精简的文件系统，它支持一些简单的操作和一些抽象操作，例如，排序和通知。

- **丰富**

Zookeeper的原语操作是很丰富的，可实现一些协调数据结构和协议。例如，分布式队列、分布式锁和一组同级别节点中的“领导者选举”。

- **高可靠**

Zookeeper支持集群模式，可以很容易的解决单点故障问题。

- **松耦合交互**

不同进程间的交互不需要了解彼此，甚至可以不必同时存在，某进程在zookeeper中留下消息后，该进程结束后其它进程还可以读这条消息。

- **资源库**

Zookeeper实现了一个关于通用协调模式的开源共享存储库，能使开发者免于编写这类通用协议。

### ZooKeeper的安装

- **独立模式安装**

Zookeeper的运行环境是需要java的，建议安装oracle的java6.

可去官网下载一个稳定的版本，然后进行安装：[http://zookeeper.apache.org/](http://zookeeper.apache.org/)

解压后在zookeeper的conf目录下创建配置文件zoo.cfg，里面的配置信息可参考统计目录下的zoo_sample.cfg文件，我们这里配置为：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper-data/
clientPort=2181
```

**tickTime**：指定了ZooKeeper的基本时间单位（以毫秒为单位）；

**initLimit**：指定了启动zookeeper时，zookeeper实例中的随从实例同步到领导实例的初始化连接时间限制，超出时间限制则连接失败（以tickTime为时间单位）；

**syncLimit**：指定了zookeeper正常运行时，主从节点之间同步数据的时间限制，若超过这个时间限制，那么随从实例将会被丢弃；

**dataDir**：zookeeper存放数据的目录；

**clientPort**：用于连接客户端的端口。

- **启动一个本地的ZooKeeper实例**

```bash
% zkServer.sh start
```

检查ZooKeeper是否正在运行

```bash
echo ruok | nc localhost 2181
```

若是正常运行的话会打印“imok”。

### ZooKeeper监控

- **远程JMX配置**

默认情况下，zookeeper是支持本地的jmx监控的。若需要远程监控zookeeper，则需要进行进行如下配置。

默认的配置有这么一行：

```bash
ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY org.apache.zookeeper.server.quorum.QuorumPeerMain"
```

咱们在**$JMXLOCALONLY**后边添加jmx的相关参数配置：

```bash
ZOOMAIN="-Dcom.sun.management.jmxremote
        -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY
                -Djava.rmi.server.hostname=192.168.1.8
                -Dcom.sun.management.jmxremote.port=1911
                -Dcom.sun.management.jmxremote.ssl=false
                -Dcom.sun.management.jmxremote.authenticate=false
                 org.apache.zookeeper.server.quorum.QuorumPeerMain"
```

这样就可以远程监控了，可以用jconsole.exe或jvisualvm.exe等工具对其进行监控。

### ZooKeeper的存储模型

Zookeeper的数据存储采用的是结构化存储，结构化存储是没有文件和目录的概念，里边的目录和文件被抽象成了节点（node），zookeeper里可以称为znode。Znode的层次结构如下图：

![ZooKeeper层次结构](https://onekook.me/bower_components/extend/images/ZooKeeper-01.png)

最上边的是根目录，下边分别是不同级别的子目录。

### Zookeeper客户端的使用

- **zkCli.sh**

可使用**./zkCli.sh -server localhost**来连接到Zookeeper服务上。

使用**ls /**可查看根节点下有哪些子节点，可以双击Tab键查看更多命令。

- **Java客户端**

可创建org.apache.zookeeper.ZooKeeper对象来作为zk的客户端，注意，java api里创建zk客户端是异步的，为防止在客户端还未完成创建就被使用的情况，这里可以使用同步计时器，确保zk对象创建完成再被使用。

- **C客户端**

可以使用zhandle_t指针来表示zk客户端，可用zookeeper_init方法来创建。可在ZK_HOME\src\c\src\ cli.c查看部分示例代码。

### Zookeeper创建Znode

Znode有两种类型：短暂的和持久的。短暂的znode在创建的客户端与服务器端断开（无论是明确的断开还是故障断开）连接时，该znode都会被删除；相反，持久的znode则不会。

```java
public class CreateGroup implements Watcher{
    private static final int SESSION_TIMEOUT = 1000;//会话延时

    private ZooKeeper zk = null;
    private CountDownLatch countDownLatch = new CountDownLatch(1);//同步计数器

    public void process(WatchedEvent event) {
        if(event.getState() == KeeperState.SyncConnected){
            countDownLatch.countDown();//计数器减一
        }
    }

    /**
     * 创建zk对象
     * 当客户端连接上zookeeper时会执行process(event)里的countDownLatch.countDown()，计数器的值变为0，则countDownLatch.await()方法返回。
     * @param hosts
     * @throws IOException
     * @throws InterruptedException
     */
    public void connect(String hosts) throws IOException, InterruptedException {
        zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);
        countDownLatch.await();//阻塞程序继续执行
    }

    /**
     * 创建group
     * 
     * @param groupName 组名
     * @throws KeeperException
     * @throws InterruptedException
     */
    public void create(String groupName) throws KeeperException, InterruptedException {
        String path = "/" + groupName;
        String createPath = zk.create(path, null, Ids.OPEN_ACL_UNSAFE/*允许任何客户端对该znode进行读写*/, CreateMode.PERSISTENT/*持久化的znode*/);
        System.out.println("Created " + createPath);
    }

    /**
     * 关闭zk
     * @throws InterruptedException
     */
    public void close() throws InterruptedException {
        if(zk != null){
            try {
                zk.close();
            } catch (InterruptedException e) {
                throw e;
            }finally{
                zk = null;
                System.gc();
            }
        }
    }
}
```

这里我们使用了同步计数器CountDownLatch，在connect方法中创建执行了zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);之后，下边接着调用了CountDownLatch对象的await方法阻塞，因为这是zk客户端不一定已经完成了与服务端的连接，在客户端连接到服务端时会触发观察者调用process()方法，我们在方法里边判断一下触发事件的类型，完成连接后计数器减一，connect方法中解除阻塞。

还有两个地方需要注意：这里创建的znode的访问权限是open的，且该znode是持久化存储的。

测试类如下：

```java
public class CreateGroupTest {
    private static String hosts = "192.168.1.8";
    private static String groupName = "zoo";
    
    private CreateGroup createGroup = null;
    
    /**
     * init
     * @throws InterruptedException 
     * @throws KeeperException 
     * @throws IOException 
     */
    @Before
    public void init() throws KeeperException, InterruptedException, IOException {
        createGroup = new CreateGroup();
        createGroup.connect(hosts);
    }
    
    @Test
    public void testCreateGroup() throws KeeperException, InterruptedException {
        createGroup.create(groupName);
    }
    
    /**
     * 销毁资源
     */
    @After
    public void destroy() {
        try {
            createGroup.close();
            createGroup = null;
            System.gc();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

由于zk对象的创建和销毁代码是可以复用的，所以这里我们把它分装成了接口：

```java
/**
 * 连接的观察者，封装了zk的创建等
 * @author leo
 *
 */
public class ConnectionWatcher implements Watcher {
    private static final int SESSION_TIMEOUT = 5000;

    protected ZooKeeper zk = null;
    private CountDownLatch countDownLatch = new CountDownLatch(1);

    public void process(WatchedEvent event) {
        KeeperState state = event.getState();
        
        if(state == KeeperState.SyncConnected){
            countDownLatch.countDown();
        }
    }
    
    /**
     * 连接资源
     * @param hosts
     * @throws IOException
     * @throws InterruptedException
     */
    public void connection(String hosts) throws IOException, InterruptedException {
        zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);
        countDownLatch.await();
    }
    
    /**
     * 释放资源
     * @throws InterruptedException
     */
    public void close() throws InterruptedException {
        if (null != zk) {
            try {
                zk.close();
            } catch (InterruptedException e) {
                throw e;
            }finally{
                zk = null;
                System.gc();
            }
        }
    }
}
```
