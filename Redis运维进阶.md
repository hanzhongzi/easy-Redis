## Redis的备份与恢复

**Redis自动备份有两种方式**
第一种是通过dump.rdb文件实现备份
第二种使用aof文件实现自动备份

均是备份到当时启动redis的所在目录

**dump.rdb备份**
Redis服务默认的自动文件备份方式(AOF没有开启的情况下)，在服务启动时，就会自动从dump.rdb文件中去加载数据。
**具体配置在redis.conf
save 900 1
save 300 10
save 60 10000**
也可以手工执行save命令实现手动备份

```
127.0.0.1:6379> set name key
OK
127.0.0.1:6379> SAVE
OK
127.0.0.1:6379> set name key1
OK
127.0.0.1:6379> BGSAVE
Background saving started
```

redis快照到dump文件时，会自动生成dump.rdb的文件

```
# The filename where to dump the DB
dbfilename dump.rdb
-rw-r--r-- 1 root root   253 Apr 17 20:17 dump.rdb
```

**SAVE命令**表示使**用主进程**将当前数据库快照到dump文件
**BGSAVE命令**表示，主进程会**fork一个子进程**来进行快照备份
两种备份不同之处，前者会阻塞主进程，后者不会。

**恢复举例**

```
# Note that you must specify a directory here, not a file name.dir 
/usr/local/redisdata/
#备份文件存储路径
127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/usr/local/redisdata"
127.0.0.1:6379> set key 001
OK
127.0.0.1:6379> set key1 002
OK
127.0.0.1:6379> set key2 003
OK
127.0.0.1:6379> save
OK
```

**将备份文件备份到其它目录**

```
[root@test ~]# ll /usr/local/redisdata/
total 4
-rw-r--r-- 1 root root 49 Apr 17 21:24 dump.rdb

[root@test ~]# date
Tue Apr 17 21:25:38 CST 2018
[root@test ~]# cp ./dump.rdb /tmp/
```

**删除数据**

```
127.0.0.1:6379> del key1
(integer) 1
127.0.0.1:6379> get key1
(nil)
```

**关闭服务，将原备份文件拷贝回save备份目录**

```
[root@test ~]# redis-cli -a foobared shutdown
[root@test ~]# lsof -i :6379
[root@test ~]# cp /tmp/dump.rdb /usr/local/redisdata/
cp: overwrite ‘/usr/local/redisdata/dump.rdb’? y
[root@test ~]# redis-server /usr/local/redis/redis.conf &
[1] 31487
```

**登录查看数据是否恢复**

```
[root@test ~]# redis-cli -a foobared
127.0.0.1:6379> mget key key1 key2
1) "001"
2) "002"
3) "003"
```

**AOF自动备份**
redis服务默认是关闭此项配置

```
###### APPEND ONLY MODE ##########
appendonly no

# The name of the append only file (default: "appendonly.aof")
appendfilename "appendonly.aof"

# appendfsync always
appendfsync everysec

# appendfsync no
```

配置文件的相关参数，前面已经详细介绍过。
AOF文件备份，是备份所有的历史记录以及执行过的命令，和mysql binlog很相似，在恢复时就是重新执次一次之前执行的命令，需要注意的就是在恢复之前，和数据库恢复一样需要手工删除执行过的del或误操作的命令。

> **AOF与dump备份不同**
> 1、aof文件备份与dump文件备份不同
> 2、服务读取文件的优先顺序不同，会按照以下优先级进行启动
>   如果只配置AOF,重启时加载AOF文件恢复数据
>   如果同时 配置了RBD和AOF,启动是只加载AOF文件恢复数据
>   如果只配置RBD,启动时将加载dump文件恢复数据

**注意：只要配置了aof，但是没有aof文件，这个时候启动的数据库会是空的**

## Redis 生产优化介绍

**1、内存管理优化** 

```
hash-max-ziplist-entries 512  
 hash-max-ziplist-value 64  
 list-max-ziplist-entries 512  
 list-max-ziplist-value 64
 #list的成员与值都不太大的时候会采用紧凑格式来存储，相对内存开销也较小
```

> **在linux环境运行Redis时，如果系统的内存比较小，这个时候自动备份会有可能失败，需要修改系统的vm.overcommit_memory 参数，这个参数是linux系统的内存分配策略**
>   0 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
>   1 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
>   2 表示内核允许分配超过所有物理内存和交换空间总和的内存
>  **Redis官方的说明是，建议将vm.overcommit_memory的值修改为1，可以用下面几种方式进行修改：**
> （1）编辑/etc/sysctl.conf 改vm.overcommit_memory=1，然后sysctl -p 使配置文件生效
> （2）sysctl vm.overcommit_memory=1
>  （3）echo 1 > /proc/sys/vm/overcommit_memory

**2、内存预分配
3、持久化机制**
  定时快照：效率不高，会丢失数据
  AOF：保持数据完整性（一个实例的数量不要太大2G最大）

> **优化总结**
> 1）根据业务需要选择合适的数据类型
> 2）当业务场景不需持久化时就关闭所有持久化方式（采用ssd磁盘来提升效率）
> 3）不要使用虚拟内存的方式，每秒实时写入AOF
> 4）不要让REDIS所在的服务器物理内存使用超过内存总量的3/5
> 5）要使用maxmemory
> 6）大数据量按业务分开使用多个redis实例





## 集群：参考https://zhuanlan.zhihu.com/p/62936527

#### 1.主从复制：

通过执行slaveof命令或设置slaveof选项，让一个服务器去复制另一个服务器的数据。被复制的服务器称为：Master主服务；对主服务器进行复制的服务器称为：Slave从服务器。主数据库可以进行读写操作，当写操作导致数据变化时会自动将数据同步给从数据库。而从数据库一般是只读的，并接受主数据库同步过来的数据。一个主数据库可以拥有多个从数据库，而一个从数据库只能拥有一个主数据库。

主从复制问题：当master down，需要手动将一台slave使用slaveof no one提升为master要实现自动，就需要redis哨兵。

##### 原理步骤：

1. 从服务器向主服务器发送SYNC命令
2. 主服务器收到SYNC命令后，执行BGSAVE命令，在后台生成RDB文件，使用缓冲区记录从现在开始执行的所有的写命令。
3. 当主服务器的BGSAVE命令执行完毕后，主服务器后将BGSAVE命令生成的RDB文件发送给从服务器，从服务器接收并载入这个RDB文件，将自己的数据库状态更新至主服务器执行BGSAVE命令时的数据库状态。
4. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器，从服务器执行这些写命令，将自己的数据库状态更新至主服务器数据库当前所处的状态。

#### 2.哨兵模式：

为了解决Redis的主从复制的不支持高可用性能，Redis实现了Sentinel哨兵机制解决方案。由一个或多个Sentinel去监听任意多个主服务以及主服务器下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线的主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已经下线的服务器，并且Sentinel可以互相监视。

当有多个Sentinel，在进行监视和转移主从服务器时，Sentinel之间会自己首先进行选举，选出Sentinel的leader来进行执行任务。



#### 3.redis集群：

集群是Redis提供的分布式数据库方案，集群通过分片来进行数据共享，并提供复制和故障转移功能。一个Redis集群通常由多个节点组成；最初，每个节点都是独立的，需要将独立的节点连接起来才能形成可工作的集群。

Cluster Nodes命令和Cluster Meet命令，添加和连接节点形成集群。

Redis中的集群分为主节点和从节点。其中主节点用于处理槽；而从节点用于复制某个主节点，并在被复制的主节点下线时，代替下线的主节点继续处理命令请求。







参考大佬的文档：[大佬文档](https://blog.csdn.net/miss1181248983/article/details/90056960)