---

title: Redis笔记
top_img: transparent
date: 2022-06-24 14:00:44
updated: 2022-06-24 14:00:44
tags:
  - Redis
  - Docker
  - C++
categories: Redis
keywords:
description: 介绍 Redis 中的数据类型原理和相关命令。
---

Redis 是一个高性能的、键值对数据库，**数据直接存储在内存中**，只有需要持久化时才写入硬盘。

## 安装启动

ubuntu 下使用 apt 安装的 redis 版本较老，可采用源码安装。

```bash
wget https://download.redis.io/releases/redis-6.2.6.tar.gz && tar xzf redis-6.2.6.tar.gz

cd redis-stable && make -j
make test
sudo make install
sudo mkdir /etc/redis && sudo cp redis.conf /etc/redis # 复制配置文件到 /etc
```

安装完成后，会有以下命令可用：

- **redis-cli**: Redis客户端
- **redis-server**: Redis服务器启动命令
- **redis-benchmark**: 性能测试工具
- **redis-check-aof**: 修复有问题的AOF文件
- **redis-check-rdb**: 修复有问题的RDB文件
- **redis-sentinel**: Redis集群使用

设置好配置文件后，启动方法为：

```bash
redis-server /etc/redis/redis.conf
```

### 配置文件

`Redis` 启动时有很多配置参数，这些参数设置均在 `redis.conf` 中，且有注释和分类，**注意：redis 配置文件中注释必须以 # 开头，且必须单独一行**。

**除了在启动前通过修改配置文件的方式修改配置，启动后也可通过`CONFIG SET/GET`命令修改、查看配置。**

`Redis` 的所有特性和功能都在配置文件中有所体现，建议仔细了解。

**NETWORK**:

```bash
bind 127.0.0.1 ::1  # 允许指定地址的host访问server，如果要全部都能访问，注释掉该行
                    # 注意，即使注释掉了bind，但protect-mode启动了的，
                    # 并且没有给server设置密码，外部仍然无法访问
protected-mode yes  # 保护模式，禁止其他主机在无密码的方式下访问本机的redis-server
port 6379           # 监听端口号
tcp-backlog         # 连接队列=未完成三次握手队列+已经完成三次握手队列，高并发环境下需要高 backlog 值
timeout 0           # 客户端N秒空闲后，关闭连接，0永不关闭
tcp-keepalive 300   # 对访问客户端的一种心跳检测，每个n秒检测一次，建议设60
unixsocket /tmp/redis.sock # 使用unixsocket连接，比 ip+port 效率更高
```

**GENERAL**:

```bash
daemonize yes       # 设置守护进程（后台启动），关闭当前终端后不会关闭redis
loglevel notice     # 日志级别，总共支持四个级别：debug、verbose、notice、warning
logfile ""          # 指定日志文件名称
databases 16        # 设置库的数量
```

**SNAPSHOTTING** 保存数据到磁盘文件 **RDB**

```bash
save 3600 1 300 100 60 10000
                    # 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可多个条件配合
rdbcompression yes  # 指定存储至本地数据库时是否压缩数据，默认为 yes，Redis 采用 LZF 压缩，
                    # 如果为了节省 CPU 时间，可以关闭该选项，但会导致数据库文件变的巨大
dbfilename dump.rdb # 指定本地数据库文件名
dir ./              # 指定本地数据库文件的存放路径
```

**REPLICATION** 同步到从机

```bash
replicaof <masterip> <masterport>
                    # 设置当本机为 slave 服务时，设置 master 服务的 IP 地址及端口，
                    # 在 Redis 启动时，它会自动从 master 进行数据同步
masterauth <master-password>
                    # 当 master 服务设置了密码保护时，slave 服务连接 master 的密码
```

**SECURITY**:

```bash
requirepass 123456  # 设置server密码，默认不带密码！！！
```

**CLIENTS**:

```bash
maxclients 120      # 最多连接的客户端数目，设置同一时间最大客户端连接数，默认无限制，
                    # Redis 可同时打开的客户端连接数为Redis进程可打开的最大文件描述符数
```

**MEMORY**:

```bash
maxmemory <bytes>   # 指定 Redis 最大内存限制，Redis 数据都在内存中，达到最大内存后，
                    # Redis 会先尝试清除部分key，清除策略由 maxmemory-policy 指定
                    # 若无法有效清除，将无法再进行写入操作，但仍然可以进行读取操作。
                    # Redis 新的 vm 机制，会把 Key 存放内存，Value 会存放在 swap 区
maxmemory-policy noeviction
                    # volatile-lru：使用 LRU 算法移除 key，只对设置了过期时间的键；
                    # allkeys-lru：在所有集合 key 中，使用 LRU 算法移除 key
                    # LRU是淘汰最长时间没有被使用的，LFU是淘汰一段时间内，使用次数最少的
                    # volatile-lfu：使用 LFU 算法移除 key，只对设置了过期时间的键；
                    # allkeys-lfu：在所有集合 key 中，使用 LFU 算法移除 key
                    # volatile-random：在过期集合中移除随机的 key，只对设置了过期时间的键
                    # allkeys-random：在所有集合 key 中，移除随机的 key
                    # volatile-ttl：移除那些 TTL 值最小的 key，即那些最近要过期的 key
                    # noeviction：不进行移除。针对写操作，只是返回错误信息
```

**APPEND ONLY MODE** 保存写操作到文件 **AOF**

```bash
appendonly no       # 是否启用AOF，将写操作记录到文件，以便重启时恢复数据
appendfilename appendonly.aof
                    # 指定保存操作信息的文件名
appenddirname ""    # 指定保存操作信息的文件路径
appendfsync no      # 指定更新操作记录的条件：
                    # no：表示不主动进行同步，把同步时机交给操作系统。
                    # always：始终同步，每次Redis的写入都会立刻记入日志；性能较差但数据完整性比较好
                    # everysec：表示每秒同步一次，如果宕机，本秒的数据可能丢失。
auto-aof-rewrite-percentage 100
                    # AOF 采用文件追加方式，文件会越来越大为避免出现此种情况，新增了重写机制,
                    # 当AOF 文件的大小超过所设定的阈值时，Redis 就会启动 AOF 文件的内容压缩，
                    # 只保留可以恢复数据的最小指令集
auto-aof-rewrite-min-size 64mb
                    # 设置重写的基准值，最小文件 64MB。达到这个值开始重写
```

### Docker 运行

`Redis` 官方有提供镜像，可直接`docker pull redis:6.2.6`拉取。

主要是启动时的端口和文件映射，需要根据`Redis`存储的文件路径和使用的端口确定。下面是一个例子：

```bash
# 1. 准备一个空文件夹存放redis相关的数据
mkdir redisdata && cd redisdata
# 2. 创建并编辑配置文件，下面给了一个示例
vim redis.conf
# 3. 拉取 redis 镜像
docker pull redis:6.2.6
# 4. 启动容器，下面非后台运行，使用 `-d` 后台运行
docker run --name redis -p 6379:6379 -v $(pwd):/data \
    redis:6.2.6 redis-server /data/redis.conf
```

**注意事项**：

1. **配置文件的中设置不对会造成容器启动后又马上**`Exit`，一个简单的示例如下：

    ```bash
    # bind 127.0.0.1 ::1
    # daemonize 必须设置为默认的 no
    daemonize no
    # 设置RDB备份文件目录
    dir /data/
    # 目录不存在也会导致容器自动停止
    logfile "/data/redis-server.log"
    # 设置密码
    requirepass 123456
    ```

2. 配置文件中的相关路径要有效。

3. 如果在 `Docker` 中运行 `Redis` ，并且设置 `daemonize` 为 `yes`，即将 `Redis` 进程守护化时，最终的 `Docker exec` 进程（启动 `Redis` 的那个进程）就无事可做了，所以该进程退出，容器也随之结束。[来自 stackoverflow 上的回答](https://stackoverflow.com/questions/50790197/why-redis-in-docker-need-set-daemonize-to-no)

4. 如果希望 `redis` **后台运行可以使用 `-d` 选项**，不过这样启动失败时难以查看错误的日志信息，所以可以先不带该选项测试是否可正常启动。无误后再使用`-d`选项。

5. **启动失败后，记得删除掉该容器**，再重新尝试启动，否则名字冲突导致启动失败。

也可以自制镜像，将设置好的配置文件拷贝到容器中。

```Dockerfile
FROM redis
COPY redis.conf /etc/redis/redis.conf
CMD [ "redis-server", "/etc/redis/redis.conf"]
```

## 基础操作

### redis-cli

```bash
redis-cli -h 127.0.0.1 -p 6379 -a password
#  -h <hostname>      Server hostname (default: 127.0.0.1).
#  -s <socket>        Server socket (overrides hostname and port).
#  -a <password>      Password to use when connecting to the server.
```

| cmd              | explain                           |
| :--------------- | :-------------------------------- |
| **auth** passwd  | 授权                              |
| **ping**         | 测试连通性                        |
| **shutdown**     | 关闭数据库服务进程 `redis-server` |
| **select** index | 切换数据库                        |
| **create** index | 创建数据库                        |
| **drop** index   | 删除数据库                        |

### Key

| cmd | explain |
|:----|:-----|
| **keys** * | 查看当前库的所有key |
| **exists** key | 查看key是否存在 |
| **del** key | 删除key |
| **unlink** key | 删除key，只是将key从数据库中删除，真正的删除会在后续异步操作 |
| **type** key | 查看key的类型 |
| **expire** key seconds | 设置key的过期时间 |
| **ttl** key | 查看key的过期时间 |
| **dbsize** | 查看当前库的key数量 |
| **flushdb** | 清空当前库 |
| **flushall** | 清空所有库 |

虽然 `Redis` 是 `key-value` 的方式存储数据，`key` 与 `key` 之间没有关系，但通常可以通过 `key` 的命名来提供一定的“信息”，`Redis` 的官方文档推荐使用冒号`:`作为键名的分隔符，因为它可以提高可读性和维护性，并且可以利用`SCAN`命令进行模式匹配。并且通常在使用 `GUI` 工具查看数据时，也会用`:`来分组。

```yaml
set md|dv:label:json 0
set md|dv:a94de756-b776-496b-bfe1-a6f3edd1d0cd 0
set md|dv:0f0a1014-43be-4531-95ee-34fdffc0b681 0
set md|dv:service:name:device-rest 0
set md|dv:label:float 0
```

上面的几个 `key` 在 `redis-desktop-manager` 中显示效果如下：

![image-20230307231649741](../images/Redis%E7%AC%94%E8%AE%B0/image-20230307231649741.png)

注意：是以 `:` 分割，`key`中的`|`没有实际意义。

### String

String 是二进制安全的，可以包含任何数据。如jpg图片或者序列化的对象，但最大长度512MB。

String 的数据结构为动态字符串，当字符串小于1Mb时，加倍扩容；大于1Mb时，一次扩容增加1Mb。

| cmd | explain |
|:----|:-----|
| **set** key value | 设置key的值，如果key存在，则覆盖 |
| **get** key | 查看 key 对应的值 |
| **append** key value | 在key的值后面追加value |
| **strlen** key | 查看key的长度 |
| **setnx** key value | 只有当key不存在时，才设置key的值，否则不设置 |
| **incr** key | 将key的值增加1 |
| **decr** key | 将key的值减少1 |
| **incrby** key increment | 将key的值增加increment |
| **decrby** key decrement | 将key的值减少decrement |

**批量操作的命令**：

| cmd | explain |
|:----|:-----|
| **mset** key1 value1 ... keyN valueN | 设置多个key的值 |
| **mget** key1 ... keyN | 获取多个key的值 |
| **msetnx** key1 value1 ... keyN valueN | 只有当所有key都不存在时，才设置多个key的值，一个失败则全部失败 |
| **getrange** key start end | 获取key的值的一部分 [start, end], 包含start 和end |
| **setrange** key offset value | 设置key的值的一部分 [offset, offset+value.length()], <br />如果offset+value.length()大于key的长度，则扩展key的长度 |
| **setex** key seconds value | 设置key的值并设置过期时间 |
| **getset** key value | 设置key的值，并返回key的旧值 |

### List

列表是值的结构，多个字符串的集合，底层是一个双向链表，对两端的操作性能很高，对中间的操作性能较差。

底层：quicklist，将部分数据组合为一个ziplist，多个ziplist通过前后指针组合为双向链表（quicklist），避免对单个小值如 int 分配两个前后指针，节约空间。

| cmd | explain |
|:----|:-----|
| **lpush** key value | 在key的左边添加一个值（也可同时插入多个值） |
| **rpush** key value | 在key的右边添加一个值 |
| **lrange** key start stop | 获取key的值的一部分，包含start和stop |
| **lpop** key | 删除并返回key的左边的值（值在键在，值空键消） |
| **rpop** key | 删除并返回key的右边的值 |
| **lpoplpush** source destination | 将source的左边的值移动到destination的右边 |
| **lpoprpush** source destination | 将source的左边的值移动到destination的右边 |
| **rpoprpush** source destination | 将source的右边的值移动到destination的左边 |
| **rpoplpush** source destination | 将source的右边的值移动到destination的左边 |
| **lindex** key index | 获取key的值的第index个元素 |
| **llen** key | 获取key的值的长度 |
| **linsert** key before|after pivot value | 在key的值的第index个元素前或后插入一个值 |
| **lrem** key count value | 从左边删除count个值为value的元素 |
| **lset** key index value | 设置key的值的第index个元素的值 |

### Hash

值部分也是 field-value 结构！

底层有两种：当数据较少时用zipist，当数据较多时用hashtable。

| cmd | explain |
|:----|:-----|
| **hset** key field value | 设置key的值的field的值 |
| **hget** key field | 获取key的值的field的值 |
| **hmset** key field1 value1 ... fieldN valueN | 设置key的值的多个field的值 |
| **hexists** key field | 判断key的值是否包含field |
| **hkeys** key | 获取key的值的所有field |
| **hvals** key | 获取key的值的所有value |
| **hlen** key | 获取key的值的长度 |
| **hdel** key field | 删除key的值的field |
| **hincrby** key field increment | 设置key的值的field的值加increment |
| **hsetnx** key field value | 只有在key的值的field不存在时，才设置key的值的field的值 |

### Set

string 类型的无序集合，底层为哈希表，与list类似，但是没有重复的值，每个值都是唯一的。

底层：字典，用hash实现。

| cmd | explain |
|:----|:-----|
| **sadd** key value | 将一个或多个值插入到key的集合中 |
| **smembers** key | 获取key的值的所有元素 |
| **sismember** key value | 判断key的值是否包含value |
| **scard** key | 获取key的值的长度 |
| **srem** key value | 删除key的值中的value |
| **spop** key | 随机获取并删除key的值中的一个元素 |
| **srandmember** key | 随机获取key的值中的一个元素，但不删除 |
| **smove** source destination value | 将source的值中的value移动到destination的值中 |
| **sinter** source1 ... sourceN | 获取所有source的值的交集 |
| **sunion** source1 ... sourceN | 获取所有source的值的并集 |
| **sdiff** source1 ... sourceN | 获取所有source的值的差集 |

### Zset

与set类似，但是一个有序集合，每个值都有一个score，score越小，编号越小

底层：hash + 跳跃表

| cmd | explain |
|:----|:-----|
| **zadd** key score value | 将一个或多个值插入到key的有序集合中 |
| **zrange** key start stop [withscores] | 获取key的值的一部分，包含start和stop |
| **zrangebyscore** key min max [withscores] | 获取key的值的一部分，包含min和max |
| **zrevrangebyscore** key max min [withscores] | 获取key的值的一部分，包含max和min |
| **zincrby** key increment value | 设置key的值的field的值加increment |
| **zrem** key value | 删除key指定值的field |
| **zcount** key min max | 获取指定区间的元素个数 |
| **zrank** key value | 获取key的值的field的编号，从0开始 |

### Bitmap

bitmaps 本身不是一种数据类型，就是字符串，但可以对位进行操作。可以把bitmaps想象成一个以位为单位的数组，每一位只能存储0或1，数组的下标在bitmaps中叫做偏移量offset。

| cmd | explain |
|:----|:-----|
| **setbit** key offset value  | value 只能是 0 或 1 |
| **getbit** key offset | 获取偏移位上的值 |
| **bitcount** key [start end] | 获取指定区间（未指定区间则全部）的`1`元素个数，**区间单位是字节byte 而不是 bit** |
| **bitop** operation destkey srckey1 ... srckeyN | 进行指定操作，操作类型有：and, or, xor, not |

### Geo

就是元素的二维坐标，在地图上就是经纬度。redis提供了经纬度设置、查询、范围查询、距离查询、经纬度Hash等功能。

有效的经纬度范围是：-180<=经度<=180, -85<=纬度<=85，坐标位置超过范围时，返回错误。

| cmd | explain |
|:----|:-----|
| **geoadd** key longitude latitude member [longitude latitude member ...] | 将一个或多个值插入到key的Geo中 |
| **geodist** key member1 member2 | 获取两个元素之间的距离 |
| **geopos** key member1 member2 | 获取两个元素的坐标 |
| **geodist** key member1 member2 [m/km/ft/mi] | 获取两个元素之间的距离 |
| **georadius** key longitude latitude radius [m/km/ft/mi] | 获取指定范围内的元素 |

## 事物

Redis 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。

**Redis 事务的主要作用就是串联多个命令防止别的命令插队。**

```bash
127.0.0.1:6379> multi           # 开始记录
OK
127.0.0.1:6379(TX)> set k1 v1   # 命令入队
QUEUED
127.0.0.1:6379(TX)> set k2 v1
QUEUED
127.0.0.1:6379(TX)> set k3 v3
QUEUED
127.0.0.1:6379(TX)> exec        # 执行命令
1) OK
2) OK
3) OK
127.0.0.1:6379> get k3          # 查看事物执行情况
"v3"
```

从输入 Multi 命令开始，输入的命令都会依次进入命令队列中，但不会执行，直到输入 Exec 后，Redis 会将之前的命令队列中的命令依次执行。组队的过程中可以通过 discard 来放弃组队。

### 错误

错误可能发生在**组队阶段**，或者**执行阶段**。

- **组队中某个命令出现了报告错误，执行时整个的所有队列都会被取消。**
- **如果执行阶段某个命令报出了错误，则只有报错的命令不会被执行，而其他的命令都会执行，不会回滚。**

### 事务三特性

1. **单独的隔离操作**
    事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
2. **没有隔离级别的概念**
    队列中的命令没有提交之前都不会实际被执行，因为事务提交前任何指令都不会被实际执行
3. **不保证原子性**
    事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

## hiredis/redis++

异步操作的命令，或使用了回调函数的操作没有给出例子。

### redis-cli 建立连接

```c++
#include <sw/redis++/redis++.h>

sw::redis::Redis redis("tcp://127.0.0.1:6379");
sw::redis::Redis redis("tcp://127.0.0.1:6379", "password");
```

### Key ops

| cmd | `redis++`                                          |
|:----|:-----|
| **keys** * | `std::vector<std::string> keys = redis.keys("*");` |
| **exists** key | `bool exists = redis.exists("key");` |
| **del** key | `int deleted = redis.del("key");` |
| **unlink** key | `int unlinked = redis.unlink("key");` |
| **type** key | `sw::redis::RedisType type = redis.type("key");` |
| **expire** key seconds | `bool success = redis.expire("key", 60);` |
| **ttl** key | `long long ttl = redis.ttl("key");` |
| **dbsize** | `long long size = redis.dbsize();` |
| **flushdb** | `bool success = redis.flushdb();` |
| **flushall** | `bool success = redis.flushall();` |

### String ops

| cmd | `redis++`                                     |
|:----|:-----|
| **set** key value | `bool success = redis.set("key", "value");` |
| **get** key | `std::string value = redis.get("key");` |
| **append** key value | `int length = redis.append("key", "value");` |
| **strlen** key | `int length = redis.strlen("key");` |
| **setnx** key value | `bool success = redis.setnx("key", "value");` |
| **incr** key | `long long result = redis.incr("key");` |
| **decr** key | `long long result = redis.decr("key");` |
| **incrby** key increment | `long long result = redis.incrby("key", 10);` |
| **decrby** key decrement | `long long result = redis.decrby("key", 10);` |

**批量操作的命令**：

| cmd | explain |
|:----|:-----|
| **mset** key1 value1 ... keyN valueN | `bool success = redis.mset({"key1", "value1", "key2", "value2"});` |
| **mget** key1 ... keyN | `std::vector<std::string> values = redis.mget({"key1", "key2"});` |
| **msetnx** key1 value1 ... keyN valueN | `bool success = redis.msetnx({"key1", "value1", "key2", "value2"});` |
| **getrange** key start end | `std::string sub = redis.getrange("key", 1, 3);` |
| **setrange** key offset value | `int length = redis.setrange("key", 1, "new");` |
| **setex** key seconds value | `bool success = redis.setex("key", 60, "value");` |
| **getset** key value | 设置key的值，并返回key的旧值 |

### List ops

| cmd | explain |
|:----|:-----|
| **lpush** key value | `int length = redis.lpush("list", "element1", "element2");` |
| **rpush** key value | `int length = redis.rpush("list", "element1", "element2");` |
| **lrange** key start stop | `std::vector<std::string> elements = redis.lrange("list", 0, -1);` |
| **lpop** key | `std::string element = redis.lpop("list");` |
| **rpop** key | `std::string element = redis.rpop("list");` |
| **lpoplpush** source destination | `bool success = redis.lpoplpush("source", "destination");` |
| **lpoprpush** source destination | `bool success = redis.lpoprpush("source", "destination");` |
| **rpoprpush** source destination | `bool success = redis.rpoprpush("source", "destination");` |
| **rpoplpush** source destination | `bool success = redis.rpoplpush("source", "destination");` |
| **llen** key | `int64_t len = redis.llen("mylist");` |
| **lindex** key index | 获取key的值的第index个元素 |
| **linsert** key before|after pivot value | 在key的值的第index个元素前或后插入一个值 |
| **lrem** key count value | 从左边删除count个值为value的元素 |
| **lset** key index value | 设置key的值的第index个元素的值 |

### Hash ops

| cmd | explain |
|:----|:-----|
| **hset** key field value | `bool success = redis.hset("hash", "field", "value");` |
| **hget** key field | `std::string value = redis.hget("hash", "field");` |
| **hmset** key field1 value1 ... fieldN valueN | `bool success = redis.hmset("hash", {{"field1", "value1"}, {"field2", "value2"}});` |
| **hexists** key field | `bool exists = redis.hexists("hash", "field");` |
| **hkeys** key | `std::vector<std::string> fields = redis.hkeys("hash");` |
| **hvals** key | `std::vector<std::string> values = redis.hvals("hash");` |
| **hlen** key | `int count = redis.hlen("hash");` |
| **hdel** key field | `bool success = redis.hdel("hash", "field");` |
| **hincrby** key field increment | `int value = redis.hincrby("hash", "field", 1);` |
| **hsetnx** key field value | `bool success = redis.hsetnx("hash", "field", "value");` |

### Set ops

| cmd | explain |
|:----|:-----|
| **sadd** key value | `int count = redis.sadd("set", {"elem1", "elem2", "elem3"});` |
| **smembers** key | `std::vector<std::string> members = redis.smembers("set");` |
| **sismember** key value | `bool exist = redis.sismember("set", "elem");` |
| **scard** key | `int count = redis.scard("set");` |
| **srem** key value | `int count = redis.srem("set", {"elem1", "elem2"});` |
| **spop** key | `std::string elem = redis.spop("set");` |
| **srandmember** key | `std::vector<std::string> elems = redis.srandmember("set", 2);` |
| **smove** source destination value | `bool success = redis.smove("set1", "set2", "elem");` |
| **sinter** source1 ... sourceN | `std::vector<std::string> elems = redis.sinter({"set1", "set2"});` |

## 相关资料

- [redis 菜鸟教程](https://www.runoob.com/redis/redis-tutorial.html)
- [官方命令文档查询](https://redis.io/commands/)
- [Redis官方推荐的各语言的client库](https://redis.io/docs/clients/)
- [python 中通过redis-py使用redis](https://www.runoob.com/w3cnote/python-redis-intro.html)
- [【尚硅谷】Redis 6 入门到精通](https://www.bilibili.com/video/BV1Rv41177Af) 有讲义
