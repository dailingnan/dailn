---
title: redis深入学习之实现主从和哨兵
date: 2019-01-21 01:41:26
categories: "Redis 教程"
tags: [微缓存,Redis]
---

<Excerpt in index | 首页摘要> 

# redis主从架构​	

### 主从的优势

​	单机的 redis，能够承载的 QPS 大概就在上万到几万不等。对于缓存来说，一般都是用来支撑**读高并发**的。因此架构做成主从(master-slave)架构，一主多从，主负责写，并且将数据复制到其它的 slave 节点，从节点负责读。所有的**读请求全部走从节点**。这样也可以很轻松实现水平扩容，**支撑读高并发**。

<!-- more -->
<The rest of contents | 余下全文>

​	主从架构除了能够支撑更高的并发量外，还保障了可用性，有了主从，当 master 挂掉的时候，运维让从库过来接管，服务就可以继续，否则 master 需要经过数据恢复和重启的过程，这就可能会拖很长的时间，影响线上业务的持续服务。

## redis 配置主从

接下来我们通过不同的端口来模拟3台机，实现redis主从

复制redis.conf多分配置文件，并通过端口命名，完成后应该包含三个配置文件

redis-6379.conf

redis-6380.conf

redis-6381.conf

#### 配置Redis主节点

```properties
port 6381 #指定端口
daemonize yes #设置后台启动
slave-read-only yes #从节点只读
requirepass "123456" #设置密码
```

#### 配置从节点

```properties
port 6380\6379 #指定端口
daemonize yes #设置后台启动
slave-read-only yes #从节点只读
requirepass "123456" #设置密码
#区别主节点
slaveof 127.0.0.1 6381 #配置主节点信息
masterauth "123456" #连接主节点配置
```

配置完主从后，可以启动测试一下

``` ./src/redis-cli -p 6381 -a 123456 ```连接主节点客户端

输入info能看到当前节点的信息，如果是主节点还可以看到从节点的信息

```properties
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=524289,lag=1
slave1:ip=127.0.0.1,port=6379,state=online,offset=524289,lag=1
master_repl_offset:524555
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:159
repl_backlog_histlen:524397
```

主节点读写都是可以的

```properties
127.0.0.1:6381> set test 1
OK
127.0.0.1:6381> keys *
1) "test"
```

```./src/redis-cli -p 6379 -a 123456 ```连接从节点客户端

输入info能看到当前节点的信息

```properties
# Replication
role:slave
master_host:127.0.0.1
master_port:6381
master_link_status:up
```

从节点只能不能写数据

```properties
127.0.0.1:6380> get test
"3"
127.0.0.1:6380> set test 2
(error) READONLY You can't write against a read only slave.
127.0.0.1:6380> 
```

Redis主从实现好了，一旦主节点出现故障，我们可以通过`slaveof no one`手动将从节点t提升为新的主节点，保障Redis继续正常工作，由于从节前之前同步了数据，所有此时从节点的数据还是和宕机前主节点的数据一致的。



## Redis 哨兵

sentinel，中文名是哨兵。哨兵是 redis 集群机构中非常重要的一个组件，主要有以下功能：

- 集群监控：负责监控 redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

#### 哨兵的优势

​	目前我们讲的 Redis 还只是主从方案，最终一致性。可以实现高并发以及故障转移，但是缺陷就是必须**手动**切换主节点，首先不能保障任何时候Redis主节点挂了都有人在，其次，主节点挂了，也不能第一时间切换。所以Redis提供了哨兵（Sentinel），自动切换主节点。

#### 配置哨兵

​	配置哨兵只需要修改```sentinel.conf```配置文件即可，我们同样复制3份，并以端口命名

sentinel-6379.conf

sentinel-6380.conf

sentinel-6381.conf

配置```sentinel.conf``就不区分主节点和从节点了，统一配置主节点的信息就可以了，他会自动去从主节点找到从节点的信息的

```properties
port 26379\26380\26381 #配置端口
sentinel monitor mymaster 127.0.0.1 6381 2 #配置主节点信息  2代表只要有两个从节点任务主节点宕机了，主节点就客观宕机了
sentinel auth-pass mymaster 123456 #连接主节点密码
```

通过```./src/redis-sentinel sentinel-6379.conf```分别启动三个哨兵,控制台打印如下信息

```properties
18198:X 20 Jan 23:53:43.317 # Sentinel ID is 57e93a8599ad80e26046e70441dbcb83248c8323
18198:X 20 Jan 23:53:43.317 # +monitor master mymaster 127.0.0.1 6381 quorum 1
18198:X 20 Jan 23:53:43.317 # +slave slave  127.0.0.1 6379  127.0.0.1 6379 @mymaster127.0.0.1 6381 
18198:X 20 Jan 23:53:43.317 # +slave slave  127.0.0.1 6380  127.0.0.1 6380 @mymaster127.0.0.1 6381 
```

此时，我们把6381的Redis人为kill掉，观察控制台，会打印如下信息

```properties
18198:X 20 Jan 23:55:34.945 # +odown master mymaster 127.0.0.1 6381 #quorum 1/1
18198:X 20 Jan 23:58:43.681 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
18198:X 20 Jan 23:58:43.681 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:14.378 # +new-epoch 2
18198:X 21 Jan 00:00:14.378 # +try-failover master mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:14.383 # +vote-for-leader 57e93a8599ad80e26046e70441dbcb83248c8323 2
18198:X 21 Jan 00:00:14.383 # +elected-leader master mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:14.383 # +failover-state-select-slave master mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:14.449 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:14.449 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:14.501 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:15.019 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:15.019 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:15.114 # +failover-end master mymaster 127.0.0.1 6380
18198:X 21 Jan 00:00:15.114 # +switch-master mymaster 127.0.0.1 6381 127.0.0.1 6380
18198:X 21 Jan 00:00:15.115 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381

```

```switch-master mymaster 127.0.0.1 6381 127.0.0.1 6380```从这里可以看到，主节点宕机后，6380从节点自动提升成主节点

```java
18198:X 20 Jan 23:58:43.681 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.378 # +new-epoch 2
18198:X 21 Jan 00:00:14.378 # +try-failover master mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.383 # +vote-for-leader 57e93a8599ad80e26046e70441dbcb83248c8323 2
18198:X 21 Jan 00:00:14.383 # +elected-leader master mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.383 # +failover-state-select-slave master mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.449 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.449 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:14.501 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:15.019 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:15.019 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:15.114 # +failover-end master mymaster 127.0.0.1 6379
18198:X 21 Jan 00:00:15.114 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6381
18198:X 21 Jan 00:00:15.115 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:15.115 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381
18198:X 21 Jan 00:00:45.128 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
18198:X 21 Jan 00:02:16.242 # -sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
18198:X 21 Jan 00:02:26.181 * +convert-to-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381

```

```./src/redis-cli -p 6380 -a 123456 ```连接6380客户端

```properties
# Replication
role:master
connected_slaves:1
slave1:ip=127.0.0.1,port=6379,state=online,offset=524289,lag=1
master_repl_offset:524555
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:159
repl_backlog_histlen:524397
```

可以看到6380此时已经是主节点了，通过sentinel，我们就已经实现了自动切换redis主从节点

其实sentinel实现主节点切换是通过修改Redis配置文件的，此时我们再去查看redis-6380.conf配置文件，就会发现，之前的```slaveof 127.0.0.1 6381```配置已经不见了，同时redis-6379.conf的配置文件，已经由原先的```slaveof 127.0.0.1 6381```变成了```slaveof 127.0.0.1 6380```(在配置文件底部)



# 总结

通过Redis主从和哨兵结合，我们就能在实现Redis并发量增加的同时保障了Redis的可用性，以及出现故障时能够自动实现故障转移。

参考文章

[李代桃僵 —— Sentinel]: https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5afc366751882567105ff0f3

