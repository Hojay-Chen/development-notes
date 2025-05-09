# 一、介绍
## 1. 概述

Redis（Remote Dictionary Server  远程字典服务器）是一个开源的内存数据库，遵守 BSD 协议，它提供了一个高性能的**键值（key-value）存储系统**，常用于**缓存、消息队列、会话存储、分布式锁等应用场景**。Redis数据库属于**NoSQL中的一种**。

## 2. 特点

- **丰富的数据类型：**Redis 不仅仅支持简单的 key-value 类型的数据，还提供了 list、set、zset（有序集合）、hash 等数据结构的存储。这些数据类型可以更好地满足特定的业务需求，使得 Redis 可以用于更广泛的应用场景。
- **高性能的读写能力：**Redis 能读的速度是 110000次/s，写的速度是 81000次/s。这种高性能主要得益于 Redis 将数据存储在内存中，从而显著提高了数据的访问速度。
- **原子性操作：**Redis 的所有操作都是原子性的，这意味着操作要么完全执行，要么完全不执行。这种特性对于确保数据的一致性和完整性非常重要。
- **持久化机制：**Redis 支持数据的持久化，可以将内存中的数据保存在磁盘中，以便在系统重启后能够再次加载使用。这为 Redis 提供了数据安全性，确保数据不会因为系统故障而丢失。
- **丰富的特性集：**Redis 还支持 publish/subscribe（发布/订阅）模式、通知、key 过期等高级特性。这些特性使得 Redis 可以用于消息队列、实时数据分析等复杂的应用场景。
- **主从复制和高可用性：**Redis 支持 master-slave 模式的数据备份，提供了数据的备份和主从复制功能，增强了数据的可用性和容错性。
- **支持 Lua 脚本：**Redis 支持使用 Lua 脚本来编写复杂的操作，这些脚本可以在服务器端执行，提供了更多的灵活性和强大的功能。
- **单线程模型：**尽管 Redis 是单线程的，但它通过高效的事件驱动模型来处理并发请求，确保了高性能和低延迟。



# 二、数据类型

## 1. string

### 1.1 底层实现原理

Redis中的String类型底层实现主要基于SDS (Simple Dynamic String简单动态字符串)结构，并结合int、embstr、raw 等不同的编码方式进行优化存储。

SDS底层结构是sdshdr,在redis 4.x及以上版本引入了sdshdr变种，如sdshdr16、sdshdr64 等,根据字符串长度动态选择不同的实现，进一步优化内存使用。

**SDS结构**

```c
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

- len (长度) :记禄SDS字符串数组的长度, 当需要获取字符串长度的时候，只需要返回这个成员变量的值就可以了时间复杂度是0(1)。

- alloc (分配空间长度) :这 个字段的主要作用是指分配给字符数组的存储的空间大小，当需要计算剩余空间大小的时候，只需要alloc - len就可以直接进行计算,然后判断空间大小是否符合修改需求,如果不满足需求的话，就执行相应的修改操作,这样的话就可以很好地解决我们上面所说的缓冲区溢出问题。

- flags (示SDS的类型) -共设计了五种类型的SDS,分别是sdshdr 5、sdshdr 8、sdshdr 16、sdshdr 32、sdshdr 64 (这个的记忆也很简单，就是32开始，128, 即2的多少次方去记忆就可以了)，通过使用不同存储类型的结构题， 灵活保存不同大小的字符串,从而节省内存空间。

- buf (存储数据的字符数组) :主要起到保存数据的作用,如字符串、二进制数据(二 进制安全就是一个重要原因)等。

**不同的编码选择**

- int编码：如果一个字符串可以被解析为整数，并且整数值比较小，Redis会直接使用整数编码。

  ```c
  struct redisObject {
      unsigned type:4;      // 数据类型（字符串、哈希等）
      unsigned encoding:4;  // 编码类型（int、embstr、raw等）
      int64_t ptr;          // 实际的数据指针，这里直接存储整数值
  };
  ```

  比如直接存储123，则：

  ![redis_string_nvodabo](.\images\redis_string_nvodabo.png)

- embstr编码：当字符串长度比较短（小于等于44字节），Redis会使用embstr编码，这种编码将所有的字符串相关结构体和字符串数据存放在连续的内存块中，分配内存的时候，只需要分配一次，减少内存分配和管理的开销。【

  ```c
  struct redisObject {
      unsigned type:4;       // 数据类型
      unsigned encoding:4;   // 编码类型，这里是 embstr
      void *ptr;             // 指向 sdshdr 结构
  };
  
  struct sdshdr {
      uint32_t len;          // 当前字符串长度
      uint32_t alloc;        // 已分配的内存大小
      unsigned char flags;   // 编码类型
      char buf[];            // 实际字符串数据
  };
  ```

  ![redis_string_onqbf](.\images\redis_string_onqbf.png)

- row编码：当字符串长度超过44字节时，Redis会使用raw编码，这种编码方式将结构体和实际字符串数据分开存储，以便处理更长的数据。

  > redis4.0及之后的版本，这个界限是44，前面版本是39。

  ```c
  struct redisObject {
      unsigned type:4;       // 数据类型
      unsigned encoding:4;   // 编码类型，这里是 raw
      void *ptr;             // 指向 sdshdr 结构
  };
  
  struct sdshdr {
      uint32_t len;          // 当前字符串长度
      uint32_t alloc;        // 已分配的内存大小
      unsigned char flags;   // 编码类型
      char buf[];            // 实际字符串数据
  };
  ```

  ![redis_string_qpnicna](.\images\redis_string_qpnicna.png)

### 1.2 指令

#### 1.2.1 `SET`

```bash
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```

**功能**：将值`value`关联到键`key`。

**参数**：

- `key`：键名。
- `value`：值。
- `EX seconds`：设置键的过期时间为`seconds`秒。
- `PX milliseconds`：设置键的过期时间为`milliseconds`毫秒。
- `NX`：仅在键不存在时设置键。
- `XX`：仅在键已存在时设置键。

**示例**：

```bash
SET "hello" "world" EX 10
```

#### 1.2.2 `GET`

```bash
GET key
```

**功能**：获取键`key`的值。

**参数**：

- `key`：键名。

**返回值**：

- 如果键存在，返回键的值。
- 如果键不存在，返回`nil`。

**示例**：

```bash
GET "hello"
```

#### 1.2.3 `INCR`/`DECR`

```bash
INCR key
DECR key
```

**功能**：

- `INCR key`：将键`key`的值加1。
- `DECR key`：将键`key`的值减1。

**参数**：

- `key`：键名。

**返回值**：

- 返回操作后的值。

**示例**：

```bash
INCR "counter"
DECR "counter"
```

#### 1.2.4 `MSET`/`MGET`

```bash
MSET key1 value1 key2 value2 ...
MGET key1 key2 ...
```

**功能**：

- `MSET`：同时设置多个键值对。
- `MGET`：同时获取多个键的值。

**参数**：

- `key1`, `value1`, `key2`, `value2`...：键值对。
- `key1`, `key2`...：键名。

**返回值**：

- `MGET`返回一个数组，包含每个键的值（如果键不存在，则返回`nil`）。

**示例**：

```bash
MSET "name" "Alice" "age" "25"
MGET "name" "age"
```

### 1.3 应用

#### 1.3.1 分布式锁

##### 1.3.1.1 锁的获取

**设计**

使用`SET`命令获取锁，确保操作的原子性：

```java
boolean acquireLock(Jedis jedis, String lockKey, String uuid, int expireTime) {
    String result = jedis.set(lockKey, uuid, "NX", "PX", expireTime);
    return "OK".equals(result);
}
```

- **`lockKey`**：锁的键名，用于唯一标识锁。
- **`uuid`**：当前线程的唯一标识符，用于区分不同线程。
- **`expireTime`**：锁的过期时间（以毫秒为单位）。
- **`NX`**：确保只有在键不存在时才设置成功。
- **`PX`**：设置锁的过期时间。

**设计原因**

1. **原子性**：`SET`命令的`NX`选项确保了只有在锁不存在时才能设置成功，避免了多个进程同时获取锁的问题。
2. **唯一标识符**：使用UUID作为锁的值，确保每个进程的锁标识符是唯一的，防止误释放其他进程的锁。
3. **过期时间**：设置过期时间（`PX`）可以防止锁永久占用。如果持有锁的进程崩溃或超时未释放锁，锁会自动过期，其他进程可以重新获取锁。

##### 1.3.1.2 锁的释放

**设计**

使用Lua脚本释放锁，确保操作的原子性：

```java
boolean releaseLock(Jedis jedis, String lockKey, String uuid) {
    String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end";
    Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(uuid));
    return Long.valueOf(result.toString()) == 1;
}
```

**设计原因**

1. **原子性**：Lua脚本在Redis中是原子执行的，避免了在检查锁值和删除锁键之间的时间窗口问题。
2. **安全性**：只有当锁的值与当前进程的UUID一致时，才删除锁键，防止误释放其他进程的锁。

##### 1.3.1.3 锁的续期

**设计**

如果操作时间可能超过锁的过期时间，需要定期续期锁：

```java
boolean renewLock(Jedis jedis, String lockKey, String uuid, int expireTime) {
    String result = jedis.set(lockKey, uuid, "PX", expireTime);
    return "OK".equals(result);
}
```

在操作过程中，定期调用上述方法更新锁的过期时间。

**设计原因**

1. **防止锁过期**：如果操作时间较长，可能会导致锁提前过期，其他进程可能会误获取锁。通过定期续期，可以确保锁在操作完成前不会过期。
2. **安全性**：续期时仍然使用UUID作为锁的值，确保只有当前进程可以续期自己的锁。

##### 1.3.1.4 锁的重试机制

**设计**

如果获取锁失败，使用指数退避算法重试：

```java
boolean tryAcquireLock(Jedis jedis, String lockKey, String uuid, int expireTime, int maxRetries, double baseDelay) {
    for (int attempt = 0; attempt < maxRetries; attempt++) {
        if (acquireLock(jedis, lockKey, uuid, expireTime)) {
            return true;
        }
        double delay = baseDelay * Math.pow(2, attempt) + Math.random() * 0.1;
        try {
            Thread.sleep((long) (delay * 1000));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    return false;
}
```

**设计原因**

1. **避免冲突**：在高并发场景下，多个进程可能同时尝试获取锁。通过指数退避算法，可以减少冲突的概率。
2. **随机性**：在每次重试时加入随机延迟，可以进一步减少多个进程同时重试的概率，避免“同步”问题。
3. **限制重试次数**：设置最大重试次数，避免无限重试导致资源浪费。

##### 1.3.1.5 锁的可重入性

**设计**

如果需要支持可重入锁，可以使用额外的键来记录锁的持有次数：

- **获取锁**：

  ```java
  long acquireReentrantLock(Jedis jedis, String lockKey, String uuid, String reentrantKey, int expireTime) {
      long count = jedis.incr(reentrantKey);
      if (count == 1) {
          acquireLock(jedis, lockKey, uuid, expireTime);
      }
      return count;
  }
  ```

- **释放锁**：

  ```java
  long releaseReentrantLock(Jedis jedis, String lockKey, String uuid, String reentrantKey) {
      long count = jedis.decr(reentrantKey);
      if (count == 0) {
          releaseLock(jedis, lockKey, uuid);
      }
      return count;
  }
  ```

- **锁的持有次数键**：

  ```java
  String reentrantKey = lockKey + ":count";
  ```

**设计原因**

1. **可重入性**：支持一个进程多次获取同一把锁，避免死锁。
2. **计数机制**：通过`INCR`和`DECR`命令记录锁的持有次数，确保只有在锁的持有次数为0时才释放锁。
3. **独立性**：使用独立的键（`reentrantKey`）来记录持有次数，避免与锁的值（`uuid`）混淆。



### 1.4 优势

- **常数时间复杂度的字符串长度获取**：通过`len`字段可以直接获取字符串长度，而C字符串需要遍历整个字符串来计算长度。
- **空间分配优化**：SDS会预先分配额外的空间（`alloc - len`），减少字符串扩展时的内存分配次数。
- **二进制安全**：SDS可以存储任意二进制数据，而C字符串以`'\0'`作为结束标志，无法存储包含`'\0'`的二进制数据。
- **兼容性**：SDS保留了C字符串的大部分特性，方便在Redis内部使用。



## 2. hash

### 2.1 底层实现原理

Hash底层结构需要分成两种情况：

- Redis6及之前，Hash数据结构底层是通过hashtable + Ziplist来实现的。

- Redis7之后，Hash数据结构底层是通过hashtable + Listpack来实现的。

> Ziplist和Listpack查找key的效率是类似的，时间复杂度都是0 (n) ，其主要区别就在于Listpack 解决了Ziplist 的级联更新问题。

Redis内有两个值，分别是hash-max-ziplist-entries ( hash-max-listpack-entries )以及hash-max-ziplist-value ( hash-max-listpack-value )，即Hash类型键的字段个数(默认512) 以及每个字段名和字段值的长度(默认64)。这两个值是可以修改的，通过如下指令进行修改：

```
config set hash-max-ziplist-entries 4399  # 更改为4399
config set hash-max-ziplist-value 2024  # 更改为2024
```

> redis 7.0为了兼容早期的版本，没有把Ziplist相关的值删掉。

- 当hash小于这两个值的时候，会使用Listpack或者Ziplist 进行存储。
- 当大于这两个值的时候会使用hashtable 进行存储。

这里需要注意一个点， 在使用hashtable结构之后，就不会再退化成Ziplist或Listpack，之后都是使用hashtable进行存储。

> #### Ziplist实现原理
>
> Ziplist (压缩列表)是一种紧凑的数据结构，它将所有元素紧密排列存储在单个连续内存块中，十分节省空间。这些元素可以是字符串或整数，且每个元素的存储都紧凑地排列在一起。
>
> 以下是Ziplist的具体结构：
>
> ![redis_Ziplist_obgabs](.\images\redis_ziplist_obgabs.png)
>
> - zlbytes: 记录整个Ziplist所占用的字节数。
>- zltail: 记录Ziplist中最后一个节点距离Ziplist起始地址的偏移量。
> - entry：各个节点的数据。
> - zllen: 记录Ziplist中节点的个数。
> - zlend: 特殊值0xFF,用于标记Ziplist的结束。
> 
> entry的结构如下,会记录前一个节点的长度和编码：
>
> ![redis_Ziplist_nonfas](.\images\redis_ziplist_nonfas.png)
>
> 因为entry需要记录前一个元素的大小，如果前面插入的元素很大，则已经存在的entry的pre_ entry_length字段需要变大，它一旦变大后续的节点也需要变，所以可能导致级联更新的情况，影响性能。
>查询需按顺序遍历所有元素,逐个检查是否匹配查询条件。
> 
> 
>
> #### Listpack实现原理
>
> Listpack采用一种紧凑的格式来存储多个元素(本质上仍是一个字节数组) ，并组使用多种编码方式来表示不同长度的数据。
>
> Listpack的结构如下：
>
> ![redis_Listpack_zbnojabs](.\images\redis_listpack_zbnojabs.png)
>
> - header: 整个Listpack的元数据，包括总长度和总元素个数。
>- elements: 实际存储的元素，每个元素包括长度和数据部分。
> - end: 标识Listpack结束的特殊字节。
> 
> element的内部结构如下：
>
> ![redis_Listpack_zanofdna](.\images\redis_listpack_zanofdna.png)
>
> - encoding-type: 元素的编码类型。
>- element-data: 实际存放的数据。
> - element-tot-len: encoding-type + element-data的总长度，包含自己的长度。
> 
> 之所以设计它来替换Ziplist就是因为Ziplist连锁更新的问题，因为Ziplist的每个entry会记录之前的entry长度。
>
> 如果前面的entry长度变大，那么当前entry记录前面entry的字段所需的空间也需要扩大，而当前的大了，可能后面的entry也得大,这就是所谓的连锁更新，比较影响性能。
>
> 而Listpack的每个元素，仅记录自己的长度，这样一来修改会新增不会影响后面的长度变大，也就避免了连锁更新的问题。
>
> 
>
> #### hashtable实现原理
>
> hashtable就是哈希表实现，查询时间复杂度为0(1)，效率非常快。hashtable的结构如下：
>
> ```c
>typedef struct dictht {
>     //哈希表数组
>     dictEntry **table;
>     //哈希表大小
>     unsigned long size;  
>     //哈希表大小掩码，用于计算索引值
>     unsigned long sizemask;
>     //该哈希表已有的节点数量
>     unsigned long used;
> } dictht;
> ```
> 
> dictht一共有4个字段：
>
> - table: 哈希表实现存储元素的结构,可以看成是哈希节点(dictEntry) 组成的数组。
>- size: 表示哈希表的大小。
> - sizemask: 这个是指哈希表大小的掩码，它的值永远等于size-1, 这个属性和哈希值一起约定了哈希节点所处的哈希表的位置,索引的值index = hash (哈希值) & sizemask。
> - used: 表示已经使用的节点数量。
> 
> ![redis_hashtable_aojdq](.\images\redis_hashtable_aojdq.png)
>
> 我们再看下哈希节点(dictEntry) 的组成，它主要由三个部分组成,分别为key、value 和指向下一个哈希节点的指针，其源码中结构体的定义如下：
>
> ```c
>typedef struct dictEntry {
>     //键值对中的键
>     void *key;
> 
>     //键值对中的值
>       union {
>         void *val;
>         uint64_t u64;
>         int64_t s64;
>         double d;
>     } v;
>     //指向下一个哈希表节点，形成链表
>     struct dictEntry *next;
> } dictEntry;
> ```
> 
> 结合以上图片，我们可以发现，哈希节点中的value值是由一个联合体组成的。因此,这个值可以是指向实际值的指针、64位无符号整数、64 为整数以及double的值。这样设计的主要目的就是为了节省Redis的内存空间，当值是整数或者浮点数的时候，可以将值的数据存在哈希节点中，不需要使用一个指针指向实际的值, 从一定程度上节省了空间。
>
> 实际上，redis的hash为了实现渐进式rehash，它的结构中包含了两个dictht，结构如下：
>
> ![redis_hashtable_xaosbno](.\images\redis_hashtable_xaosbno.png)
>
> **渐进式rehash**
>在平时，插入数据的时候，所有的数据都会写入ht[0]即哈希表1, ht [1]哈希表2此时就是一-张没有分配空间的空表。
> 但是随着数据越来越多，当dict的空间不够的时候，就会触发扩容条件,扩容流程主要分为三步:
> 
> 1. 首先，为哈希表2即分配空间。新表的大小是第一 个大于等于原表2倍used的2次方幂。举个例子，如果原表即哈希表1的值是1024, 那个其扩容之后的新表大小就是2048。分配好空间之后，此时dict就有了两个哈希表了，然后此时字典的rehashidx即rehash索引的值从-1暂时变成0 ，然后便开始数据转移操作。
>2. 数据开始实现转移。每次对hash进行增删改查操作，都会将当前rehashidx 的数据从哈希表1迁移到2上，然后rehashidx+1，所以迁移的过程是分多次、渐进式地完成。
>    注意:插入数据会直接插入到2表中。
> 3. 随着操作不断执行，最终哈希表1的数据都会被迁移到2中，这时候进行指针对象进行互换，即哈希表2变成新的哈希表1,而原先的哈希表1成哈希表2并且设置为空表，最后将rehashidx的值设置为-1。
> 
> 就这样，渐进式rehash的过程就完成了。



### 2.2 指令

#### 2.2.1 `HSET`

```bash
HSET key field value [field value ...]
```

**功能**：为哈希表`key`中的字段`field`设置值`value`。

**参数**：

- `key`：哈希表的键名。
- `field`：字段名。
- `value`：字段的值。

**示例**：

```bash
HSET "user:1001" "name" "Alice" "age" "25"
```

#### 2.2.2 `HGET`

```bash
HGET key field
```

**功能**：获取哈希表`key`中字段`field`的值。

**参数**：

- `key`：哈希表的键名。
- `field`：字段名。

**返回值**：

- 如果字段存在，返回字段的值。
- 如果字段不存在，返回`nil`。

**示例**：

```bash
HGET "user:1001" "name"
```

#### 2.2.3 `HGETALL`

```bash
HGETALL key
```

**功能**：获取哈希表`key`中所有的字段和值。

**参数**：

- `key`：哈希表的键名。

**返回值**：

- 返回一个数组，包含所有的字段和值。

**示例**：

```bash
HGETALL "user:1001"
```

#### 2.2.4 `HDEL`

```bash
HDEL key field [field ...]
```

**功能**：删除哈希表`key`中的一个或多个字段。

**参数**：

- `key`：哈希表的键名。
- `field`：字段名。

**返回值**：

- 返回被删除的字段数量。

**示例**：

```bash
HDEL "user:1001" "age"
```



### 2.3 应用

#### 2.3.1 用户信息存储

**设计**：

使用`HSET`命令存储用户信息，将用户的所有属性存储在一个Hash对象中：

```java
jedis.hset("user:1001", "name", "Alice");
jedis.hset("user:1001", "age", "25");
jedis.hset("user:1001", "email", "alice@example.com");
```

使用`HGET`命令获取用户信息：

```java
String name = jedis.hget("user:1001", "name");
String age = jedis.hget("user:1001", "age");
```

使用`HDEL`命令删除用户信息：

```java
jedis.hdel("user:1001", "age");
```

**设计原因**：

1. **高效存储**：将用户的所有属性存储在一个Hash对象中，节省内存。
2. **快速访问**：通过字段名可以直接访问对应的值，时间复杂度为O(1)。
3. **灵活性**：可以动态添加、删除和修改用户信息。



### 2.4 优势

- **高效的数据存储**：可以将多个字段和值存储在一个键下，节省内存。
- **快速访问**：通过字段名可以直接访问对应的值，时间复杂度为O(1)。
- **灵活性**：可以动态添加、删除和修改字段。



## 3. List

### 3.1 底层实现原理

Redis中的List数据结构底层是通过Ziplist+LinkedList或QuickList来实现的，在Redis3.2之前采用Ziplist+LinkedList实现，在Redis3.2之后只采用QuickList。当List中的元素较少时，使用压缩列表存储，节省内存；当元素较多时，自动转换为双向链表。

**Ziplist+LinkedList**

创建新列表时 redis 默认使用 redis_encoding_Ziplist 编码， 当以下任意一个条件被满足时， 列表会被转换成 redis_encoding_linkedlist 编码：

- 试图往列表新添加一个字符串值，且这个字符串的长度超过 server.list_max_ziplist_value （默认值为 64 ）。
- Ziplist 包含的节点超过 server.list_max_ziplist_entries （默认值为 512 ）。

注意：这两个条件是可以修改的，在 redis.conf 中：

```text
list-max-ziplist-value 64 
list-max-ziplist-entries 512 
```

> #### 双向链表linkedlist
>
> 当链表entry数据超过512、或单个value 长度超过64，底层就会转化成linkedlist编码。
> linkedlist是标准的双向链表，Node节点包含prev和next指针，可以进行双向遍历；还保存了 head 和 tail 两个指针，因此，对链表的表头和表尾进行插入的复杂度都为 (1) —— 这是高效实现 LPUSH 、 RPOP、 RPOP、LPUSH 等命令的关键。

**QuickList**

QuickList结合了Ziplist和双端链表的有典，每个QuickList节点都是一个Ziplist，它限制了单个Ziplist的大小，降低级联更新产生的影响。



![redis_quicklist_xasbnojc](.\images\redis_quicklist_xasbnojc.png)





### 3.2 指令

#### 3.2.1 `LPUSH`

```bash
LPUSH key value [value ...]
```

**功能**：将一个或多个值`value`插入到列表`key`的头部。

**参数**：

- `key`：列表的键名。
- `value`：要插入的值。

**返回值**：

- 返回插入操作后，列表的长度。

**示例**：

```bash
LPUSH "mylist" "world"
LPUSH "mylist" "hello"
```

#### 3.2.2 `RPUSH`

```bash
RPUSH key value [value ...]
```

**功能**：将一个或多个值`value`插入到列表`key`的尾部。

**参数**：

- `key`：列表的键名。
- `value`：要插入的值。

**返回值**：

- 返回插入操作后，列表的长度。

**示例**：

```bash
RPUSH "mylist" "end"
```

#### 3.2.3 `LPOP`

```bash
LPOP key
```

**功能**：移除并返回列表`key`的头部元素。

**参数**：

- `key`：列表的键名。

**返回值**：

- 返回被移除的元素。
- 如果列表为空，返回`nil`。

**示例**：

```bash
LPOP "mylist"
```

#### 3.2.4 `RPOP`

```bash
RPOP key
```

**功能**：移除并返回列表`key`的尾部元素。

**参数**：

- `key`：列表的键名。

**返回值**：

- 返回被移除的元素。
- 如果列表为空，返回`nil`。

**示例**：

```bash
RPOP "mylist"
```

### 3.3 应用

#### 3.3.1 消息队列

**设计**：

使用`LPUSH`和`RPOP`命令实现一个简单的消息队列：

```java
// 生产者
jedis.lpush("message_queue", "message1");
jedis.lpush("message_queue", "message2");

// 消费者
String message = jedis.rpop("message_queue");
```

**设计原因**：

1. **高效存储**：可以存储多个消息，节省内存。
2. **快速插入和删除**：在List的头部或尾部插入和删除元素的时间复杂度为O(1)。
3. **灵活性**：可以动态添加、删除和修改消息。



### 3.4 优势

- **高效的数据存储**：可以存储多个元素，节省内存。
- **快速插入和删除**：在List的头部或尾部插入和删除元素的时间复杂度为O(1)。
- **灵活性**：可以动态添加、删除和修改元素。



## 4. Set

### 4.1 底层实现原理

Redis中的Set数据结构底层是通过哈希表或整数集合（IntSet）来实现的。当集合中的所有元素都是整数且元素数量较少时，使用整数集合（IntSet）存储，节省内存；当元素较多或包含非整数元素时，自动转换为哈希表。

### 4.2 指令

#### 4.2.1 `SADD`

```bash
SADD key member [member ...]
```

**功能**：将一个或多个成员`member`添加到集合`key`中。

**参数**：

- `key`：集合的键名。
- `member`：要添加的成员。

**返回值**：

- 返回被添加的成员数量。

**示例**：

```bash
SADD "myset" "apple"
SADD "myset" "banana" "cherry"
```

#### 4.2.2 `SMEMBERS`

```bash
SMEMBERS key
```

**功能**：获取集合`key`中的所有成员。

**参数**：

- `key`：集合的键名。

**返回值**：

- 返回一个数组，包含集合中的所有成员。

**示例**：

```bash
SMEMBERS "myset"
```

#### 4.2.3 `SREM`

```bash
SREM key member [member ...]
```

**功能**：从集合`key`中移除一个或多个成员`member`。

**参数**：

- `key`：集合的键名。
- `member`：要移除的成员。

**返回值**：

- 返回被移除的成员数量。

**示例**：

```bash
SREM "myset" "banana"
```

### 4.3 应用

#### 4.3.1 用户权限管理

**设计**：

使用`SADD`命令为用户添加权限，使用`SMEMBERS`命令获取用户的所有权限，使用`SREM`命令移除用户权限：

```java
// 为用户添加权限
jedis.sadd("user:1001:permissions", "read");
jedis.sadd("user:1001:permissions", "write", "delete");

// 获取用户的所有权限
Set<String> permissions = jedis.smembers("user:1001:permissions");

// 移除用户权限
jedis.srem("user:1001:permissions", "delete");
```

**设计原因**：

1. **高效存储**：可以存储多个唯一权限，节省内存。
2. **快速查找**：通过哈希表实现，查找权限的时间复杂度为O(1)。
3. **灵活性**：可以动态添加、删除和修改权限。
4. **集合操作**：支持交集、并集、差集等集合操作，方便进行权限管理。

### 4.4 优势

- **高效的数据存储**：可以存储多个唯一元素，节省内存。
- **快速查找**：通过哈希表实现，查找元素的时间复杂度为O(1)。
- **灵活性**：可以动态添加、删除和修改元素。
- **集合操作**：支持交集、并集、差集等集合操作。



## 5. ZSet（Sorted Set）

### 5.1 底层实现原理

Redis中的ZSet (有序集合, Sorted Set)是底层由SkipList + hashtable或Ziplist实现。 ZSet 结合了集合(Set) 的特性和排序功能，能够存储具有唯一性的成员, 并根据成员的分数(score) 进行排序。

当Zset满足如下条件时，Redis 会使用压缩列表(Ziplist)来节省内存：

- 元素个数 ≤ zset-max- ziplist-entries (默认128字节)
- 元素成员名和分值的长度 ≤ zset-max ziplist-value (默认64字节)

如果任何一个条件不满足，Zset 将使用SkipList+hashtable作为底层实现：

- Skiplist：用于存储数据的排序和快速查找。

- hashtable：用于存储成员与分数的映射，提供快速通过成员查找其对应的分数。

> #### Skiplist实现原理
>
> Redis的跳表相对于普通的跳表多了一个回退指针，且score可以重复。
>
> 我们先来看一下Redis中关于跳表实现的源码：
>
> ```c
> typedef struct zskiplistNode {
>     //Zset 对象的元素值
>     sds ele;
>     //元素权重值
>     double score;
>     //后退指针
>     struct zskiplistNode *backward;
>   
>     //节点的level数组，保存每层上的前向指针和跨度
>     struct zskiplistLevel {
>         struct zskiplistNode *forward;
>         unsigned long span;
>     } level[];
> } zskiplistNode;
> ```
>
> 这里我们分析一下每个字段的含义
>
> - ele：这里用到了Redis字符串底层的一个实现sds, 主要作用是用来存储数据。
> - score: 节点的分数, double即浮点型数据。
> - backward：我们可以看到其是zskiplistNode结构体指针类型,即代表指向前一 个跳表节点
> - level：这个就是zskiplistNode的结构体数组了，数组的索引代表层级索引，这里注意与hashtable中的结构进行区分,那个使用的是联合体。
> - forward：代表下一个跳转的跳表节点，注意这里的下一个节点是指同一层的下一个节点。
> - span：主要作用代表距离下一个节点的步数。
>
> 到这里分析就差不多了，为了方便大家理解，这里放一个相关的图，下图的红色箭头就代表回退指针。
>
> Redis跳表的实现大致如下图所示了：
>
> ![redis_skiplist_znjoba](.\images\redis_skiplist_znjoba.png)
>
> **Skiplist的功能流程**
>
> - 查询元素：这里我们与传统的链表进行对比，来了解跳表查询的高效。
>
>   假设我们要查找50这个元素，如果通过传统链表的话(看最底层绿色的查询路线)，需要查找4次，查找的时间复杂度为0(n)。
>
>   但如果使用跳表的话，期需要从最上面的10开始，首先跳到40，发现目标元素比40大，然后对比后一个元素比 70小。是就前往下一层进行查找，然后40的下一个50刚好符合目标，就直接返回就可以了，这个过程的跳转次数是3次，即10 -> 40 (项层) -> 40 (第二层) -> 50(第二层)，其流程如下图所标：
>
>   ![redis_skiplist_qnivda](.\images\redis_skiplist_qnivda.png)
>
>   跳表的平均时间查询复杂度是0 (logn) ，最差的时间复杂度是0 (n) 。
>
> - 插入元素: 我们插入一条score为48的数据
>
>   - 先需要定位到第一个比score大的数据。如图所示，一下子就可以定位到50了，这里和查询的过程(上文所示)是一样的。
>   - 在定位到对应节点之后，具体是在当前节点创建数据还是增加一个层级这个是随机的，这里假设设置为2层。
>   - 定位层级后，再将每层的链表节点进行补齐，就是在40与50之间插入一个新的链表节点48,插入过程与链表插入是一样的。
>
>   最终实现的效果如下图所示：
>
>   ![redis_skiplist_zifanis](.\images\redis_skiplist_zifanis.png)



### 5.2 指令

#### 5.2.1 `ZADD`

```bash
ZADD key score member [score member ...]
```

**功能**：将一个或多个成员`member`及其分数`score`添加到有序集合`key`中。

**参数**：

- `key`：有序集合的键名。
- `score`：成员的分数。
- `member`：要添加的成员。

**返回值**：

- 返回被添加的成员数量。

**示例**：

```bash
ZADD "myzset" 1 "apple"
ZADD "myzset" 2 "banana" 3 "cherry"
```

#### 5.2.2 `ZRANGE`

```bash
ZRANGE key start stop [WITHSCORES]
```

**功能**：获取有序集合`key`中指定范围内的成员。

**参数**：

- `key`：有序集合的键名。
- `start`：起始索引。
- `stop`：结束索引。
- `WITHSCORES`：可选参数，返回成员及其分数。

**返回值**：

- 返回一个数组，包含指定范围内的成员及其分数（如果指定了`WITHSCORES`）。

**示例**：

```bash
ZRANGE "myzset" 0 -1 WITHSCORES
```

#### 5.2.3 `ZREM`

```bash
ZREM key member [member ...]
```

**功能**：从有序集合`key`中移除一个或多个成员`member`。

**参数**：

- `key`：有序集合的键名。
- `member`：要移除的成员。

**返回值**：

- 返回被移除的成员数量。

**示例**：

```bash
ZREM "myzset" "banana"
```

### 5.3 应用

#### 5.3.1 领域排行榜

**设计**：

使用`ZADD`命令为用户添加分数，使用`ZRANGE`命令获取排行榜，使用`ZREM`命令移除用户：

```java
// 为用户添加分数
jedis.zadd("leaderboard", 100, "user:1001");
jedis.zadd("leaderboard", 200, "user:1002");
jedis.zadd("leaderboard", 150, "user:1003");

// 获取排行榜
Set<String> topUsers = jedis.zrange("leaderboard", 0, -1);

// 移除用户
jedis.zrem("leaderboard", "user:1001");
```

**设计原因**：

1. **高效存储**：可以存储多个用户及其分数，节省内存。
2. **快速查找**：通过哈希表实现，查找用户的时间复杂度为O(1)。
3. **有序性**：通过跳跃表维护用户的有序性，支持范围查询。
4. **灵活性**：可以动态添加、删除和修改用户及其分数。



### 5.4 优势

- **高效的数据存储**：可以存储多个成员及其分数，节省内存。
- **快速查找**：通过哈希表实现，查找成员的时间复杂度为O(1)。
- **有序性**：通过跳跃表维护成员的有序性，支持范围查询。
- **灵活性**：可以动态添加、删除和修改成员及其分数。



## 6. Bitmap

### 6.1 底层实现原理

Redis中的Bitmap数据结构底层是通过字符串（String）来实现的。每个Bitmap键对应一个字符串，字符串中的每个字节可以表示一个位（bit），每个位可以是0或1。

**结构组成**：

- **键（Key）**：唯一标识一个Bitmap对象。
- **位（Bit）**：Bitmap中的每个位，可以是0或1。

**优势**：

- **高效的数据存储**：可以存储多个位，节省内存。
- **快速访问**：通过位偏移量可以直接访问对应的位，时间复杂度为O(1)。
- **灵活性**：可以动态设置、获取和修改位。

### 6.2 指令

#### 6.2.1 `SETBIT`

bash

复制

```bash
SETBIT key offset value
```

**功能**：将字符串`key`中偏移量为`offset`的位设置为`value`。

**参数**：

- `key`：Bitmap的键名。
- `offset`：位偏移量。
- `value`：位的值，0或1。

**返回值**：

- 返回设置前的位值。

**示例**：

bash

复制

```bash
SETBIT "mybitmap" 0 1
SETBIT "mybitmap" 1 0
```

#### 6.2.2 `GETBIT`

bash

复制

```bash
GETBIT key offset
```

**功能**：获取字符串`key`中偏移量为`offset`的位值。

**参数**：

- `key`：Bitmap的键名。
- `offset`：位偏移量。

**返回值**：

- 返回位的值，0或1。

**示例**：

bash

复制

```bash
GETBIT "mybitmap" 0
```

#### 6.2.3 `BITCOUNT`

bash

复制

```bash
BITCOUNT key [start end]
```

**功能**：计算字符串`key`中值为1的位的数量。

**参数**：

- `key`：Bitmap的键名。
- `start`：可选参数，范围的起始偏移量。
- `end`：可选参数，范围的结束偏移量。

**返回值**：

- 返回值为1的位的数量。

**示例**：

bash

复制

```bash
BITCOUNT "mybitmap"
```

### 6.3 应用

#### 6.3.1 用户签到系统

**设计**：

使用`SETBIT`命令记录用户每天的签到状态，使用`GETBIT`命令查询用户的签到状态，使用`BITCOUNT`命令统计用户的签到天数：

java

复制

```java
// 记录用户签到
jedis.setbit("user:1001:sign", 0, true); // 第一天签到
jedis.setbit("user:1001:sign", 1, true); // 第二天签到

// 查询用户签到状态
boolean isSigned = jedis.getbit("user:1001:sign", 0);

// 统计用户签到天数
long signDays = jedis.bitcount("user:1001:sign");
```

**设计原因**：

1. **高效存储**：可以存储多个位，节省内存。
2. **快速访问**：通过位偏移量可以直接访问对应的位，时间复杂度为O(1)。
3. **灵活性**：可以动态设置、获取和修改位。

------

## 7. HyperLogLog

### 7.1 底层实现原理

Redis中的HyperLogLog数据结构是一种用于基数统计的近似算法。它通过一个固定大小的内存（通常为12KB）来估计一个集合中不同元素的数量，误差率约为0.81%。

**结构组成**：

- **键（Key）**：唯一标识一个HyperLogLog对象。
- **数据结构**：内部使用一个固定大小的内存来存储基数统计信息。

**优势**：

- **高效的数据存储**：使用固定大小的内存，节省内存。
- **快速统计**：可以快速估计集合中不同元素的数量。
- **灵活性**：可以动态添加元素。

### 7.2 指令

#### 7.2.1 `PFADD`

bash

复制

```bash
PFADD key element [element ...]
```

**功能**：将一个或多个元素`element`添加到HyperLogLog`key`中。

**参数**：

- `key`：HyperLogLog的键名。
- `element`：要添加的元素。

**返回值**：

- 返回1表示元素被成功添加，返回0表示元素已存在。

**示例**：

bash

复制

```bash
PFADD "visitors" "Alice"
PFADD "visitors" "Bob" "Charlie"
```

#### 7.2.2 `PFCOUNT`

bash

复制

```bash
PFCOUNT key [key ...]
```

**功能**：返回一个或多个HyperLogLog`key`中不同元素的数量。

**参数**：

- `key`：HyperLogLog的键名。

**返回值**：

- 返回不同元素的数量。

**示例**：

bash

复制

```bash
PFCOUNT "visitors"
```

#### 7.2.3 `PFMERGE`

bash

复制

```bash
PFMERGE destkey sourcekey [sourcekey ...]
```

**功能**：将多个HyperLogLog`sourcekey`合并到一个HyperLogLog`destkey`中。

**参数**：

- `destkey`：目标HyperLogLog的键名。
- `sourcekey`：源HyperLogLog的键名。

**返回值**：

- 无返回值。

**示例**：

bash

复制

```bash
PFMERGE "total_visitors" "visitors1" "visitors2"
```

### 7.3 应用

#### 7.3.1 网站访问统计

**设计**：

使用`PFADD`命令记录网站的访问者，使用`PFCOUNT`命令统计访问者的数量，使用`PFMERGE`命令合并多个HyperLogLog：

java

复制

```java
// 记录访问者
jedis.pfadd("visitors", "Alice");
jedis.pfadd("visitors", "Bob", "Charlie");

// 统计访问者数量
long visitorCount = jedis.pfcount("visitors");

// 合并多个HyperLogLog
jedis.pfmerge("total_visitors", "visitors1", "visitors2");
```

**设计原因**：

1. **高效存储**：使用固定大小的内存，节省内存。
2. **快速统计**：可以快速估计集合中不同元素的数量。
3. **灵活性**：可以动态添加元素。

------

## 8. Geospatial



## 9. streams



## 10. modules



# 三、Redis keys 命令
DEL key 该命令用于在 key 存在时删除 key。
DUMP key 序列化给定 key ，并返回被序列化的值。
EXISTS key 检查给定 key 是否存在。
EXPIRE key seconds 为给定 key 设置过期时间，以秒计。
EXPIREAT key timestamp EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。
6	PEXPIRE key milliseconds 设置 key 的过期时间以毫秒计。
7	PEXPIREAT key milliseconds-timestamp 设置 key 过期时间的时间戳(unix timestamp) 以毫秒计
8	KEYS pattern 查找所有符合给定模式( pattern)的 key 。
9	MOVE key db 将当前数据库的 key 移动到给定的数据库 db 当中。
10	PERSIST key 移除 key 的过期时间，key 将持久保持。
11	PTTL key 以毫秒为单位返回 key 的剩余的过期时间。
12	TTL key 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
13	RANDOMKEY 从当前数据库中随机返回一个 key 。
14	RENAME key newkey 修改 key 的名称
15	RENAMENX key newkey 仅当 newkey 不存在时，将 key 改名为 newkey 。
16	[SCAN cursor MATCH pattern] [COUNT count] 迭代数据库中的数据库键。
17	TYPE key 返回 key 所储存的值的类型。
keys *: 查看当前库所有的key

exists key: 判断某个key是否存在，可以一次性输入多个key，返回实际存在的key的数量

type key: 查看key的类型

del key : 删除key,可以一次性输入多个key，返回删除成功的key的数量

unlink key：非阻塞删除，仅仅将keys从keyspace元数据中删除，真正的删除会在后续异步中操作。

ttl key：查看还有多少秒过期，-1表示永不过期，-2表示已过期

expire key 秒钟: 为给定的key设置过期时间

move key dbindex: 将当前数据库的key移动到给定的数据库当中，一个Redis默认有16个数据库，dbindex范围 [0-15]，默认使用下标为0的库

select dbindex: 切换数据库，默认为0

dbsize：查看当前数据库key的数量

flushdb: 清空当前库

flushall: 清空所有库



# 四、Redis发布订阅模式

## 1. 介绍
Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。

Redis 客户端可以订阅任意数量的频道。

## 2. Redis的发布和订阅
客户端可以订阅频道，如下图：

![redis_pubsub_xnonda](.\images\redis_pubsub_xnonda.png)

当给这个频道发布消息后，消息就会发送给订阅的客户端，如下图：

![redis_pubsub_xnoas](.\images\redis_pubsub_xnoas.png)

## 3. 发布订阅命令行实现
（1）打开一个客户端订阅channel1
```
SUBSCRIBE channel1
```

![redis_pubsub_pajsjf](.\images\redis_pubsub_pajsjf.png)

（2）打开另一个客户端，给channel1发布消息hello
```
publish channel1 hello
```

![redis_pubsub_nvhsjs](.\images\redis_pubsub_nvhsjs.png)

返回的1是订阅者数量

（3）打开第一个客户端可以看到发送的消息

![redis_pubsub_njdkd](.\images\redis_pubsub_njdkd.png)



# 五、Redis事务和锁机制



# 六、Redis持久化

Redis提供两种主要的持久化机制：

- RDB (Redis Database)快照：

  - RDB通过**生成某一时刻的数据快照**来实现持久化的，可以在特定时间间隔内保存数据的快照。

  - 适给灾难恢复和备份，能生成紧凑的二进制文件，但可能会在崩溃时丢失最后一次快照之 后的数据。

- AOF (Append Only File)日志：

  - AOF 通过将每个写操作追加到日志文件中实现持久化，支持将所有写操作记录下来以便恢复。

  - 数据恢复更为精确，文件体积较大，写时可能会消耗更多资源。

- Redis 4.0新增了RDB和AOF的混合持久化机制。

## 1. RDB

### 1.1 概述

RDB持久化通过创建快照来获取内存某个时间点上的副本，利用快照可以进行方便地进行主从复制，默认快照文件为dump.rdb。

redis.conf文件可以配置在x秒内如果至少有y个key发生变化就会触发命令进行持久化操作。

### 1.2 原理

#### 1.2.1 save

由主进程生成RDB文件，因此生成其间，主进程**无法执行正常的读写命令**，需要等待RDB结束。通过手动调用指令触发。

#### 1.2.2 bgsave

bgsave是Redis默认的备份指令，由配置文件中进行频率配置，不会阻塞正常的读写命令。当然也可以通过指令手动触发。

![redis_RDB_noknas](.\images\redis_RDB_noknas.png)

1. **检查子进程是否存在AOF/RDB的子进程正在进行**，如果有返回错误。

2. **父进程触发RDB持久化**：

   - 当Redis配置为RDB持久化模式时，父进程会定期触发RDB持久化操作。这可以通过配置文件中的`save`指令来设置，例如`save 900 1`表示如果900秒内至少有1个键被修改，则触发RDB持久化。

   - 也可以通过命令`BGSAVE`手动触发RDB持久化。

3. **创建子进程**：

   - 父进程通过`fork()`系统调用创建一个子进程，在创建子进程期间父进程是阻塞的，无法响应指令的。在`fork()`调用时，子进程会复制父进程的内存空间，但由于操作系统的写时复制（COW）机制，此时并不会真正复制内存内容，而是共享相同的物理内存页，如下图：

     ![redis_RDB_cnoan](.\images\redis_RDB_cnoan.png)

   - 只有当父进程或子进程对内存进行写操作时，才会触发COW机制，真正复制内存页，如下图：

     ![redis_RDB_csnka](.\images\redis_RDB_csnka.png)

4. **子进程负责IO操作**：

   - 子进程负责将当前内存中的数据快照写入到一个新的RDB文件中。这个过程包括：
     - 遍历内存中的所有数据。
     - 将数据序列化为RDB文件格式。
     - 将序列化后的数据写入到磁盘上的RDB文件中。
   - 由于子进程在写入RDB文件时可能会触发COW机制，因此父进程的内存状态会被复制，但不会影响父进程的正常运行。

5. **父进程继续接收指令**：

   - 在子进程进行RDB持久化的同时，父进程继续接收和处理客户端的命令。
   - 如果父进程接收到修改数据的命令，这些修改会直接作用于父进程的内存空间。由于COW机制，这些修改不会影响子进程正在写入的RDB文件。

### 1.3 优点

- 快速加载: RDB生成的快照文件是压缩的进制文件，适合备份和灾难恢复。
- 低资源占用: RDB持款化在Redis主线程之外进行，不会对主线程产生太大影响。

### 1.4 缺点

- 数据丢失风险：由于RDB是间隔性保存的快照，如果Redis崩溃，可能会丢失上次保存快照之后的数据。

### 1.5 注意

- Fork 操作会产生短暂的阻塞，微秒级别操作过后，不会阻塞主进程，整个过程不是完全的非阻塞。
- RDB于是快照备份所有数据，而不是像AOF一样存写命令，因此Redis实例重启后恢复数据的速度可以得到保证，大数据量下比AOF会快很多。
- Fork 操作利用了写时复制，类似与CopyOnWriteArrayList。



## 2. AOF

### 2.1 概述

**以日志的形式来记录每个写操作（增量保存）**，将Redis执行过的所有写指令记录下来(**读操作不记录**)， **只许追加文件但不可以改写文件**，redis启动之初会读取该文件重新构建数据，换言之，redis 重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。

### 2.2 原理

**启动**

在redis配置文件中，找到appendonly字段，将其值改为yes（默认值为no）。

```
appendonly yes
```

**持久化流程**

1. 客户端的请求写命令会被append追加到AOF缓冲区内。

2. AOF缓冲区根据AOF持久化策略[always,everysec,no]将操作sync同步到磁盘的AOF文件中，默认采用everysec策略。通过配置文件中的appendfsync参数来进行配置：

   ```
   appendfsync everysec
   ```

   appendfsync always：始终同步，每次Redis的写入都会立刻记入日志；性能较差但数据完整性比较好。

   appendfsync everysec：每秒同步，每秒记入日志一次，如果宕机，本秒的数据可能丢失。

   appendfsync no：redis不主动进行同步，把同步时机交给操作系统。

3. AOF文件大小超过重写策略或手动重写时，会对AOF文件rewrite重写，压缩AOF文件容量。

4. Redis服务重启时，会重新load加载AOF文件中的写操作达到数据恢复的目的。

![redis_aof_ksanocns](.\images\redis_aof_ksanocns.png)

**重写机制**

AOF文件随着写操作的增加会不断变大，过大的AOF文件会导致恢复速度变慢，并消耗大量磁盘空间。所以，Redis提供了AOF重写机制，即对AOF文件进行压缩。

AOF重写并不是对现有的AOF文件进行修改，而是根据当前每个键的最新之转换为对应的写命令，写入新的AOF文件，形成一个新文件。

AOF重写流程：

1. 创建子进程：Redis使用BGREWRITEAOF命令创建一个子进程，负责AOF重写操作。
2. 生成新的AOF文件：子进程根据当前数据库的状态，将每个键的最新之转换为对应的写命令，并写入新的AOF文件。
3. 处理新写入的命令：在重写过程中，主进程仍然处理新的写操作。为了避免数据不一致，主进程会将这些新的写命令同时追加到现有的AOF文件和一个缓冲区（aof_rewrite_buf）中。
4. 合并新命令：当子进程完成新的AOF文件的写入后，主进程会将缓冲区中的新命令追加到新的AOF文件中，确保其包含所有最新的操作。
5. 替换旧的AOF文件：最后，Redis使用新的AOF文件替换旧的文件，实现AOF文件的重写。

![redis_aof_csnoina](.\images\redis_aof_csnoina.png)

AOF重写触发方式：

- 手动触发：使用BGREWRITEAOF命令可以手动触发AOF重写。
- 自动触发：通过配置文件中的参数控制自动触发条件，参数如下：
  - auto-aof-rewrite-min-size：AOF文件达到该大小时允许重写（默认64MB）。
  - auto-aof-rewrite-percentage：当前AOF文件大小相对于上次重写后的增长百分比达到该值时触发重写。

7.0之前的AOF重写有三大问题:

1. 内存开销：aof_ buf 和aof_ rewrite_ _buf 中大部分内容是复的。_
2. _CPU开销：主进程需要花费CPU时间往aof rewrite_ buf 写入数据，拘子进程发送aof_ rewrite_ _buf 中的数据。子进程需要消耗CPU时间将aof_ rewrite. _buf 写入新AOF文件。_
3. 磁盘开销：aof_ buf 数据会写到当前的AOF文件，aof_ rewrite_ buf 数据写到新的AOF文件，一份数据需要写两次磁盘。

针对以上问题，Redis 7.0引入了MP-AOF (Multi-Part Append Only File)机制。简单来说就是将一个AOF文件拆分成了多个文件：

- 一个基础文件(base file) ，代表数据的初始快照
- 增量文件(incremental files) ，记录自基础文件创建以来的所有写操作，可以有多个
- 基础文件和增量文件都会存放在一个单独的目录中, 并由一个清单文件(manifest file)进行统一跟踪和管理。当重写完后，仅需更新manifest文件，加入新的增量AOF文件和基础AOF文件，然后将之前的增量AOF文件和基础AOF文件标记为历史文件(会被异步删除)即可。更新完manifest就代表AOF写结束。

![redis_aof_ioasno](.\images\redis_aof_ioasno.png)

### 2.3 优点

AOF机制比RDB机制更加可靠，因为AOF文件记录了Redis执行的所有写命令，可以在每次写操作命令执行完成后都落盘存储。

### 2.4 缺点

- AOF机制生成的AOF文件比RDB文件更大，当数据集比较大时，AOF文件会比RDB文件占用更多的磁盘空间。
- AOF机制对于数据恢复的时间比RDB机制更加耗时，因为要重新执行AOF文件中的所有操作命令。



## 3. 混合持久化

### 3.1 概述

RDB和AOF都有各自的缺点。如果RDB备份的频率低，那么丢的数据多。备份的频率高，性能影响大。AOF文件虽然丢数据比较少,但是恢复起来又比较耗时。因此Redis 4.0以后引入了混合持久化。

### 3.2 原理

**开启**

通过aof-use-rdb-preamble配置开启混合持久化。

**持久化流程**

当AOF重写的时候(注意混合持久化是在aof写时触发的)。它会先生成当前时间的RDB快照，将其写入新的AOF文件头部位置。这段时间主线程处理的操作命令会记录在重写缓冲区中, RDB写入完毕后将这里的增量数据追加到这个新AOF文件中，后再用新AOF文件替换旧AOF文件。

**恢复过程**

如此一来，当Redis通过AOF文件恢复数据时，会先加载RDB，然后再重新执行指令恢复后半部分的增量数据，这样就可以大幅度提高数据恢复的速度!



# 七、Redis架构

## 1. 单机模式

### 1.1 概述

![redis_xsioac](.\images\redis_xsioac.jpg)

　　单机模式顾名思义就是安装一个 Redis，启动起来，业务调用即可。例如一些简单的应用，并非必须保证高可用的情况下可以使用该模式。

### 1.2 优点

- 部署简单；
- 成本低，无备用节点；
- 高性能，单机不需要同步数据，数据天然一致性。

### 1.3 缺点

- 可靠性保证不是很好，单节点有宕机的风险。
- 单机高性能受限于 CPU 的处理能力，Redis 是单线程的。

　　单机 Redis 能够承载的 QPS（每秒查询速率）大概在几万左右。取决于业务操作的复杂性，Lua 脚本复杂性就极高。假如是简单的 key value 查询那性能就会很高。

　　假设上千万、上亿用户同时访问 Redis，QPS 达到 10 万+。这些请求过来，单机 Redis 直接就挂了。系统的瓶颈就出现在 Redis 单机问题上，此时我们可以通过**主从复制**解决该问题，实现系统的高并发。



## 2. 主从复制

### 2.1 概述

![redis_csoanb](.\images\redis_csoanb.jpg)

　　Redis 的复制（Replication）功能允许用户根据一个 Redis 服务器来创建任意多个该服务器的复制品，其中被复制的服务器为主服务器（Master），而通过复制创建出来的复制品则为从服务器（Slave）。 只要主从服务器之间的网络连接正常，主服务器就会将写入自己的数据同步更新给从服务器，从而保证主从服务器的数据相同。

　　数据的复制是单向的，只能由主节点到从节点，简单理解就是从节点只支持读操作，不允许写操作。主要是读高并发的场景下用主从架构。主从模式需要考虑的问题是：当 Master 节点宕机，需要选举产生一个新的 Master 节点，从而保证服务的高可用性。

### 2.2 原理

Redis的主从复制主要由两种数据同步方式实现，分别是全量同步和增量同步：

#### 2.2.1 全量同步

同步流程：

- 从节点发送`psync ? -1`，触发同步。
  - runid 指的是主服务器的run ID，从节点第一次同步不知道主节点ID，于是传递"?"。

  - offset为从服务器的复制进度，第一次同步值为-1。

- 主节点收到从节点的psync命令之后，发现runid没值，判断是全量同步，返回fullresync并带上主服务器的runid和当前复制进度offset，从服务器会存储这两个值。

- 主节点执行bgsave生成RDB文件，在RDB文件生成过程中，节点新接收到的写入数据的命令会存储到`replication buffer`中。

- RDB文件生成完毕后，节点将其发送给从节点，从节点清空旧数据，加载RDB的数据。

- 等到从节点中RDB文件加载完成之后，节点将`replication buffer`缓存的数据发送给从节点，从节点执行命令，保证数据的一致性。

- 待同步完毕后，主从之间会保持一个长连接, 主节点会通过这个连接将后续的写操作传递给从节点执行，来保证数据的一致。

同步流程图：

![redis_dpsan](.\images\redis_dpsan.png)

#### 2.2.2 增量同步

主从之间的网络可能不稳定，如果连接断开，主节点部分写操作未传递给从节点执行，主从数据就不一致了。

此时有一种选择是再次发起全量同步,但是全量同步数据量此较大，非常耗时。因此Redis在2.8版本引入了增量同步(psync 实就是2.8引入的命令)，仅需把连接断开其间的数据同步给从节点就好了。

同步流程：

- 从节点发送`psync runid offset`，触发同步。

- 主节点收到从节点的psync命令之后，会检查runid与自身的id是否相同：

  - 如果runid与主节点id不相同，则表示从节点是首次与主节点进行同步，需要进行全量同步。

  - 如果runid与主节点id相同，则此时根据offset判断从节点所同步到的位置的数据是否还在`repl_backlog_buffer`缓冲区中：

    > #### repl_ backlog_buffer
    >
    > 增量同步主要依靠`repl_backlog_buffer`缓冲区实现，`repl_ backlog_buffer`是一个环形缓冲区，默认大小为1m。每个主节点都会维护一个唯一的`repl_backlog_buffer`缓冲区供所有从节点进行增量同步时查看，主节点会将写入命令存到这个缓冲区中，但是大小有限，写入的命令超过1m后，会覆盖之前的数据，因为是环形写入。

    - 如果从节点发送的offset在`repl_backlog_buffer`中已经被覆盖，则需要进行全量同步。因此可以调整`repl_backlog_buffer`大小，尽量避免出现全量同步。
    - 如果`repl_backlog_buffer`中存在offset对应的数据，则主节点查找对应offset之后的命令数据，写入到replication buffer中，最终将其发送给从节点，从节点受到指令数据后，执行这些指令，一次增量同步的过程就完成了。

- 待同步完毕后，主从之间会保持一个长连接, 主节点会通过这个连接将后续的写操作传递给从节点执行，来保证数据的一致。

同步流程图：



![redis_pninkcsa](.\images\redis_pninkcsa.png)



### 2.3 优点

- Master/Slave 角色方便水平扩展，QPS 增加，增加 Slave 即可。
- 降低 Master 读压力，转交给 Slave 节点。
- 主节点宕机，从节点作为主节点的备份可以随时顶上继续提供服务。

### 2.4 缺点

- 可靠性保证不是很好，主节点故障便无法提供写入服务。
- 没有解决主节点写的压力。
- 数据冗余（为了高并发、高可用和高性能，一般是允许有冗余存在的）。
- 一旦主节点宕机，从节点晋升成主节点，需要修改应用方的主节点地址，还需要命令所有从节点去复制新的主节点，整个过程需要人工干预。
- 主节点的写能力受到单机的限制。
- 主节点的存储能力受到单机的限制。



## 3. 哨兵模式

### 3.1 概述

![redis_opabscs](.\images\redis_opabscs.jpg)

主从模式中，当主节点宕机之后，从节点是可以作为主节点顶上来继续提供服务，但是需要修改应用方的主节点地址，还需要命令所有从节点去复制新的主节点，整个过程**需要人工干预**。

于是，在 Redis 2.8 版本开始，引入了哨兵（Sentinel）这个概念，在**主从复制的基础**上，哨兵实现了**自动化故障恢复**。如上图所示，哨兵模式由两部分组成，哨兵节点和数据节点：

- 哨兵节点：哨兵节点是特殊的 Redis 节点，不存储数据；
- 数据节点：主节点和从节点都是数据节点。

Redis Sentinel 是分布式系统中监控 Redis 主从服务器，并提供主服务器下线时自动故障转移功能的模式。其中三个特性为：

- 监控(Monitoring)：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
- 提醒(Notification)：当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
- 自动故障迁移(Automatic failover)：当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作。

接下来我们了解一些 Sentinel 中的关键名词，然后系统讲解下哨兵模式的工作原理。

#### 3.1.1 定时任务

　　Sentinel 内部有 3 个定时任务，分别是：

- 每 1 秒每个 Sentinel 对其他 Sentinel 和 Redis 节点执行 `PING` 操作（监控），这是一个**心跳检测**，是失败判定的依据。
- 每 2 秒 Sentinel 之间通过 Master 节点的 channel 交换信息（Publish/Subscribe）。
- 每 10 秒每个 Sentinel 会对 Master 和 Slave 执行 `INFO` 命令，这个任务主要达到两个目的：
  - 发现 Slave 节点。
  - 确认主从关系。

#### 3.1.2 主观下线

　　所谓主观下线（Subjectively Down， 简称 SDOWN）指的是单个 Sentinel 实例对服务器做出的下线判断，即单个 Sentinel 认为某个服务下线（有可能是接收不到订阅，之间的网络不通等等原因）。

　　主观下线就是说如果服务器在给定的毫秒数之内， 没有返回 Sentinel 发送的 PING 命令的回复， 或者返回一个错误， 那么 Sentinel 会将这个服务器标记为主观下线（SDOWN）。这里判断主观下线等待的毫秒数由配置文件的`sentinel down-after-milliseconds <masterName> <timeout>`来配置，可以通过修改`<timeout>`来控制具体等待时间。

#### 3.1.3 客观下线

　　客观下线（Objectively Down， 简称 ODOWN）指的是多个 Sentinel 实例在对同一个服务器做出 SDOWN 判断，并且通过命令互相交流之后，得出的服务器下线判断，然后开启 failover。

　　只有在足够数量的 Sentinel 都将一个服务器标记为主观下线之后， 服务器才会被标记为客观下线（ODOWN）。只有当 Master 被认定为客观下线时，才会发生故障迁移。

#### 3.1.4 仲裁

　　仲裁指的是配置文件中的 `quorum` 选项。某个 Sentinel 先将 Master 节点标记为主观下线，然后会将这个判定通过 `sentinel is-master-down-by-addr` 命令询问其他 Sentinel 节点是否也同样认为该 addr 的 Master 节点要做主观下线。最后当达成这一共识的 Sentinel 个数达到前面说的 `quorum` 设置的值时，该 Master 节点会被认定为客观下线并进行故障转移。

　　`quorum` 的值一般设置为 Sentinel 个数的**二分之一加 1**，例如 3 个 Sentinel 就设置为 2。



### 3.2 原理

#### 3.2.1 心跳机制

1. 哨兵模式创建时：需要通过配置指定 Sentinel 与 Redis Master Node 之间的关系，然后 Sentinel 会向主节点发送INFO REPLICATION命令，来获取所有从节点的信息。

2. 在哨兵创建初期完成初始信息获取后，哨兵会进入持续监控阶段：
   - Sentinel与Redis Node：每个 Sentinel 节点会向主节点、从节点每**10秒一次**发送INFO REPLICATION命令来更新主从节点的状态信息。
   - Sentinel与Sentinel：基于 Redis 的订阅发布功能， 每个 Sentinel 节点会**每2秒一次**向主节点的hello 频道上发送该 Sentinel 节点对于主节点的判断以及当前 Sentinel 节点的信息 ，同时每个 Sentinel 节点也会订阅该频道， 来获取其他 Sentinel 节点的信息以及它们对主节点的判断。
   - Sentinel会向所有主从节点和其他Sentinel**每秒一次**执行一次PING操作，来判断所有主从节点和其他Sentinel的连接状态。
   - 当Master 被 Sentinel 标记为客观下线时，Sentinel 向下线的 Master 的所有 Slave 发送 INFO 命令的频率会从 10 秒一次改为每秒一次。

#### 3.2.2 判定客观下线

1. 每个 sentinel 哨兵节点每隔1s 向所有的master、slave以及其他 sentinel 节点发送一个PING命令，作用是通过心跳检测，检测主从服务器的网络连接状态

2. 如果 master 节点回复 PING 命令的时间超过 down-after-milliseconds 设定的阈值（默认30s），则这个 master 会被 sentinel 标记为主观下线，修改其 flags 状态为SRI_S_DOWN。

3. 当sentinel 哨兵节点将 master 标记为主观下线后，会向其余所有的 sentinel 发送sentinel is-master-down-by-addr消息，询问其他sentinel是否同意该master下线

   > 发送命令：sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>
   >
   > ip：主观下线的服务ip
   >
   > port：主观下线的服务端口
   >
   > current_epoch：sentinel的纪元
   >
   > runid：*表示检测服务下线状态，如果是sentinel的运行id，表示用来选举领头sentinel

4. 每个sentinel收到命令之后，会根据发送过来的 ip和port 检查自己判断的结果，回复自己是否认为该master节点已经下线了

   > 回复内容主要包含三个参数（由于上面发送的runid参数是*，这里先忽略后两个参数）
   >
   > down_state（1表示已下线，0表示未下线）
   >
   > leader_runid（领头sentinal id）
   >
   > leader_epoch（领头sentinel纪元）。

5. sentinel收到回复之后，如果同意master节点进入主观下线的sentinel数量大于等于quorum，则master会被标记为客观下线，即认为该节点已经不可用。

6. 在一般情况下，每个 Sentinel 每隔 10s 向所有的Master，Slave发送 INFO 命令。当Master 被 Sentinel 标记为客观下线时，Sentinel 向下线的 Master 的所有 Slave 发送 INFO 命令的频率会从 10 秒一次改为每秒一次。作用：发现最新的集群拓扑结构，以便快速发现是否有从节点可以被提升为新的主节点，从而完成故障转移。。

#### 3.2.3 选举领头sentinel

到现在为止，已经知道了master客观下线，那就需要一个sentinel来负责故障转移，那到底是哪个sentinel节点来做这件事呢？需要通过选举实现，具体的选举过程如下：

1. 判断客观下线的sentinel节点向其他 sentinel 节点发送 SENTINEL is-master-down-by-addr ip port current_epoch runid

   注意：这时的runid是自己的run id，每个sentinel节点都有一个自己运行时id。

2. 目标sentinel回复是否同意master下线并选举领头sentinel，选择领头sentinel的过程符合先到先得的原则。举例：sentinel1判断了客观下线，向sentinel2发送了第一步中的命令，sentinel2回复了sentinel1，说选你为领头，这时候sentinel3也向sentinel2发送第一步的命令，sentinel2会直接拒绝回复。

3. 如果当sentinel发现选自己的节点个数超过Leader最低票数(`quorum`和`Sentinel节点数/2+1`的最大值)，则该Sentinel节点选举为Leader；否则重新进行选举。 

4. 如果没有一个sentinel的票数满足条件，等一段时间，重新选举。

#### 3.2.4 故障转移

有了领头sentinel之后，下面就是要做故障转移了，故障转移的一个主要问题和选择领头sentinel问题差不多，到底要选择哪一个slaver节点来作为master呢？按照我们一般的常识，我们会认为哪个slave节点中的数据和master中的数据相识度高哪个slaver就是master了，其实哨兵模式也差不多是这样判断的，不过还有别的判断条件，详细介绍如下：

1. 在进行选择之前需要先剔除掉一些不满足条件的slaver，这些slaver不会作为变成master的备选：
   - 剔除列表中已经下线的从服务。
   - 剔除有5s没有回复sentinel的info命令的slave。
   - 剔除与已经下线的主服务连接断开时间超过 down-after-milliseconds * 10 + master宕机时长的slaver。

2. 选主过程：
   - 选择优先级最高的节点，通过从节点的配置文件中的replica-priority配置项，这个参数越小，表示优先级越高。
   - 如果第一步中的优先级相同，选择offset最大的，offset表示主节点向从节点同步数据的偏移量，越大表示同步的数据越多。
   - 如果第二步offset也相同，选择run id较小的。

#### 3.2.5 修改配置

新的master节点选择出来之后，还需要做一些事情配置的修改，如下：

1. 领头sentinel会对选出来的从节点执行slaveof no one 命令让其成为主节点。

2. 领头sentinel向别的slave发送slaveof命令，为这些节点配置主节点，使这些从节点向master复制数据。

3. 如果之前的master重新上线时，领头sentinel同样会给起发送slaveof命令，将其变成从节点。



### 3.3 优点

- 哨兵模式是基于主从模式的，所有主从模式的优点，哨兵模式都有。
- 哨兵模式可以自动切换，系统更健壮，可用性更高。
- Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。



###  3.4 缺点

- 主从切换需要时间，会丢失数据。
- 还是没有解决主节点写的压力。
- 主节点的写能力，存储能力受到单机的限制。

- 动态扩容困难复杂，对于集群，容量达到上限时在线扩容会变得很复杂。



## 4. 集群模式

### 4.1 概述

　　假设上千万、上亿用户同时访问 Redis，QPS 达到 10 万+。这些请求过来，单机 Redis 直接就挂了。系统的瓶颈就出现在 Redis 单机问题上，此时我们可以通过**主从复制**解决该问题，实现系统的高并发。

　　主从模式中，当主节点宕机之后，从节点是可以作为主节点顶上来继续提供服务，但是需要修改应用方的主节点地址，还需要命令所有从节点去复制新的主节点，整个过程需要人工干预。于是，在 Redis 2.8 版本开始，引入了**哨兵（Sentinel）**这个概念，在**主从复制的基础**上，哨兵实现了**自动化故障恢复**。

　　哨兵模式中，单个节点的写能力，存储能力受到单机的限制，动态扩容困难复杂。于是，Redis 3.0 版本正式推出 **Redis Cluster 集群**模式，有效地解决了 Redis 分布式方面的需求。Redis Cluster 集群模式具有**高可用**、**可扩展性**、**分布式**、**容错**等特性。

　　总结下来就是：读请求分配给 Slave 节点，写请求分配给 Master，数据同步从 Master 到 Slave 节点。

### 4.2 原理

#### 4.2.1 集群信息同步

Redis Cluster 采用无中心结构，Redis集群内每个节点都和其他所有节点连接，都会保存集群的完整拓扑信息，包括每个节点的ID、IP 地址、端口、负责的哈希槽范围等。Cluster 一般由多个节点组成，节点数量至少为 6 个才能保证组成完整高可用的集群，其中三个为主节点，三个为从节点。

节点之间使用Gossip协议进行状态交换，以保持集群的一致性和故障检测。每个节点会周期性地发送PING和PONG消息，交换集群信息，使得集群信息得以同步。

> **Gossip的优点**
>
> - 快速收敛: Gossip协议能够快速传播信媳，确保集群状态的迅速更新。
> - 降低网络负担:于信息是以随机节点间的对话访式传播，避免了集中式的状态查询，从而降低了网络流量。

如图所示，该集群中包含 6 个 Redis 节点，3 主 3 从，分别为 M1，M2，M3，S1，S2，S3。除了主从 Redis 节点之间进行数据复制外，所有 Redis 节点之间采用 Gossip 协议进行通信，交换维护节点元数据信息。

![redis_ocbasa](.\images\redis_ocbasa.png)

#### 4.2.2 集群分片

Redis集群会将数据分散到16384 (2 ^ 14)个哈希槽中，集群中的每个节负责一定范围的哈希槽，在Redis集群中，使用CRC16哈希算法计算键的哈希槽，以确定该键应存储在哪个节点。
集群哈希槽分片如图所标：

<img src=".\images\redis_Sacnjkn.png" alt="redis_Sacnjkn" style="zoom:67%;" />

每个节点会拥有一部分的槽位，然后对应的键值会根据其本身的key，映射到一个哈希槽中，其主要流程如下:

- 根据键值的key，按照CRC 16算法计算一个16 bit的值，然后将16 bit的值对16384进行取余运算，后得到一个对应的哈希槽编号。
- 根据每个节纷配的哈希槽区间，对应编号的数据落在对应的区间上，就能找到对应的分实例。

**注意**：redis 客户端可以访问集群中任意一台实例， 正常情况下这个实例包含这个数据。但如果槽被转移了，客户端还未来得及更新槽的信息，当前实例没有这个数据，则返回MOVED响应给客户端，将其重定向到对应的实例(Gossip集群内每个节点都会保存集群的完整拓扑信息)。

> ## 为什么是16384个槽呢？
>
> （1）首先是消息大小的考虑。
> 正常的心跳包需要带上节点完整配置数据，心跳还是比较频繁的，所以需要考虑数据包的大小，如果使用16384数据包只要2KB，如果用了65535则需要8KB。
> 实际上槽位信息使用一个帐度为16384位的数组来表示，节点拥有哪个槽位，就将对应位置的数据信息设置为1，否则为0.
> 心跳数据包就包含槽位信息如下图所示:
>
> ![redis_mnvlan](.\images\redis_mnvlan.png)
>
> 这里我们看到一个重点，即在消息头中最占空间的是myslots[CLUSTER_ SLOTS/8]。
>
> - 当槽位为65536时，这块的大小是: 65536+8+ 1024=8KB
> - 当槽位为16384时， 这块的大小是: 16384+8+ 1024=2KB
>
> 如果槽位为65536，这个ping消息的消息头就太大了，浪费带宽。
>
> （2）集群规模的考虑。
>
> 集群不太可能会扩展超过1000个节点，16384 够用且使得每个分片下的槽又不会太少。

#### 4.2.3 主从模式

　　Redis Cluster 为了保证数据的高可用性，加入了主从模式，**一个主节点对应一个或多个从节点**，主节点提供数据存取，从节点复制主节点数据备份，当这个主节点挂掉后，就会通过这个主节点的从节点选取一个来充当主节点，从而保证集群的高可用。

　　回到刚才的例子中，集群有 A、B、C 三个主节点，如果这 3 个节点都没有对应的从节点，如果 B 挂掉了，则集群将无法继续，因为我们不再有办法为 5501 ~ 11000 范围内的哈希槽提供服务。

　　所以我们在创建集群的时候，一定要为每个主节点都添加对应的从节点。比如，集群包含主节点 A、B、C，以及从节点 A1、B1、C1，那么即使 B 挂掉系统也可以继续正确工作。

　　因为 B1 节点属于 B 节点的子节点，所以 Redis 集群将会选择 B1 节点作为新的主节点，集群将会继续正确地提供服务。当 B 重新开启后，它就会变成 B1 的从节点。但是请注意，如果节点 B 和 B1 同时挂掉，Redis Cluster 就无法继续正确地提供服务了。

### 4.3 优点

- 无中心架构。
- 可扩展性，数据按照 Slot 存储分布在多个节点，节点间数据共享，节点可动态添加或删除，可动态调整数据分布。
- 高可用性，部分节点不可用时，集群仍可用。通过增加 Slave 做备份数据副本。
- 实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave 到 Master 的角色提升。

### 4.4 缺点

- 数据通过异步复制，无法保证**数据强一致性**。
- 集群环境搭建复杂，不过基于 Docker 的搭建方案会相对简单。



# 八、Redis实战问题

## 1. 缓存穿透

### 1.1 介绍

![redis_kxonas](E:\各种资料\Java开发笔记\我的笔记\images\redis_kxonas.png)

缓存穿透是指查询一个不存在的数据，由于缓存中肯定不存在,导致每次请求都直接访问数据库，增加数据负载。

攻击者可以通过构造不存在的key发起大量请求，对数据库造成很大的压力，可能会造成系统宕机。

### 1.2 解决办法

1. 防止非法请求：检查非法请求，封禁其IP以及账号，防止它再次为非作歹。

2. 缓存空值：将数据库中不存在的结果(例如空值)也缓存起来，并设置一个较短的过期时间，避免频繁查询数据库。

3. 使用布隆过滤器：使佣布隆过滤器来快速判断一个请求的数据是否存在，如果布隆过滤器判断数据不存在,则直接返回,避免查询数据库。

   > #### 布隆过滤器
   >
   > **实现原理**
   >
   > 布隆过滤器是由一个位数组和k个独立的哈希函数组成。
   >
   > 添加元素时，通过k个哈希函数将元素映射到位数组的k个位置上，将位数组中的这些位置设置为1。
   >
   > 检查元素是否存在时，同样通过k个哈希函数计算k个位置，如果位数组中的所有这些位置都是1，则说明元素可能存在；只要有一个位置为 0，就可以确定元素一定不存在。
   >
   > 例如某个key通过hash-1和hash-2两个哈希函数,定位到数组中的值都为1，则说明它存在。
   >
   > ![redis_oxnsao](E:\各种资料\Java开发笔记\我的笔记\images\redis_oxnsao.png)
   >
   > 如果布隆过滤器判断一个元素不存在集合中， 那么这个元素一定不在集合中，如果判断元素存在集合中则不一定是真的，因为哈希可能会存在冲突。因此布隆过滤器有误判的概率。
   >
   > ![redis_cposan](E:\各种资料\Java开发笔记\我的笔记\images\redis_cposan.png)
   >
   > 且它不好删除阮素，只能新增，如果想要删除，只能重建。
   >
   > **优点**
   >
   > - 效性：插入和查询操作都非常高效，时间复杂度为O(k), k为哈希函数的数量。
   > - 节省空间：相比于直接存储所有元素， 布隆过滤器大幅度减少了内存使用。
   > - 可扩展性：可以根据需要调整位数组的大小和哈希函数的数量来平衡时间和空间效率。
   >
   > **缺点**
   >
   > - 误判率：可能会误认为不存在的元素在集合中，但不会漏报(不存在的元素不会被认为存在)。
   > - 可删除：一旦插入元素,不能删除，因为无法确定哪些哈希值是由哪个元素设置的。
   > - 需要多个哈希函数:选择合适的哈希函数并保证它们独立性并不容易。
   >
   > **具体实现**
   >
   > - 使用Redis的bitmap数据结构
   >
   >   bitmap基本操作：
   >
   >   - SETBIT key offset value：将key的值在offset位置上的位设置为value (0或1) 。
   >   - GETBIT key offset：获取key的值在offset位置上的位的值(0或1)。
   >   - BITCOUNT key [start end]：计算字符串中设置为1的位的数量。
   >   - BITOP operation destkey key [key ...]：对一个或多个key进行位运算，并将结果存储在destkey中。支持的操作包括AND、OR、XOR和NOT。
   >
   >   实现布隆过滤器流程：
   >
   >   - 位图使用单个位来表示某个位置的状态，通过Redis的SETBIT key offset value 操作可以设置位图中某个偏移位置的值。
   >   - 假设有k个哈希函数H1, H2, .... Hk,对于每个新加入的元素x，通过这些哈希函数计算位置，将相应位置的比特位设置为1。
   >   - 查询元素是否存在时，计算相同的k个位置并用GETBIT key offset来判断这些位置是否都是1。
   >
   >   以下为Java中利用bitmap实现布隆过滤器的例子，供参考：
   >
   >   ```java
   >   import redis.clients.jedis.Jedis;
   >   import java.nio.charset.StandardCharsets;
   >   import java.util.BitSet;
   >   import java.util.List;
   >   import java.util.ArrayList;
   >   
   >   public class RedisBloomFilter {
   >   
   >       private static final String BLOOM_FILTER_KEY = "bloom_filter";
   >       private static final int BITMAP_SIZE = 1000000; // 位图大小
   >       private static final int[] HASH_SEEDS = {3, 5, 7, 11, 13, 17}; // 多个哈希函数的种子
   >   
   >       private Jedis jedis;
   >       private List<SimpleHash> hashFunctions;
   >   
   >       public RedisBloomFilter() {
   >           this.jedis = new Jedis("localhost", 6379);
   >           this.hashFunctions = new ArrayList<>();
   >           for (int seed : HASH_SEEDS) {
   >               hashFunctions.add(new SimpleHash(BITMAP_SIZE, seed));
   >           }
   >       }
   >   
   >       // 添加元素到布隆过滤器
   >       public void add(String value) {
   >           for (SimpleHash hashFunction : hashFunctions) {
   >               jedis.setbit(BLOOM_FILTER_KEY, hashFunction.hash(value), true);
   >           }
   >       }
   >   
   >       // 检查元素是否可能存在于布隆过滤器中
   >       public boolean mightContain(String value) {
   >           for (SimpleHash hashFunction : hashFunctions) {
   >               if (!jedis.getbit(BLOOM_FILTER_KEY, hashFunction.hash(value))) {
   >                   return false;
   >               }
   >           }
   >           return true;
   >       }
   >   
   >       // 关闭连接
   >       public void close() {
   >           jedis.close();
   >       }
   >   
   >       // 简单哈希函数
   >       public static class SimpleHash {
   >           private int cap;
   >           private int seed;
   >   
   >           public SimpleHash(int cap, int seed) {
   >               this.cap = cap;
   >               this.seed = seed;
   >           }
   >   
   >           public int hash(String value) {
   >               int result = 0;
   >               byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
   >               for (byte b : bytes) {
   >                   result = seed * result + b;
   >               }
   >               return (cap - 1) & result;
   >           }
   >       }
   >   
   >       // 测试
   >       public static void main(String[] args) {
   >           RedisBloomFilter bloomFilter = new RedisBloomFilter();
   >   
   >           // 添加元素到布隆过滤器
   >           bloomFilter.add("user1");
   >           bloomFilter.add("user2");
   >           bloomFilter.add("user3");
   >   
   >           // 检查元素是否可能存在
   >           System.out.println("Does user1 exist? " + bloomFilter.mightContain("user1")); // 输出: true
   >           System.out.println("Does user4 exist? " + bloomFilter.mightContain("user4")); // 输出: false
   >   
   >           // 关闭连接
   >           bloomFilter.close();
   >       }
   >   }
   >   ```
   >
   > - 使用RedisBloom模块
   >
   >   RedisBloom是Redis官方提供的插件，简化了布隆过滤器的实现，提供了更好的性能和更少的误判率控制。
   >   常用命令:
   >
   >   - BF.RESERVE key error_ rate capacity :创建一 个布隆过滤器，指定误判率和容量。
   >   - BF .ADD key item :向布隆过滤器中添加一个元素。
   >   - BF.EXISTS key item: 检查- 个元素是否可能存在。
   >   - RedisBloom 可以自动调整底层数据结构大小以适应不断增加的数据量,用户可以指定误判率。
   >
   >   RedisBloom实现布隆过滤器的步骤示例:
   >
   >   1. 创建布隆过滤器(纡RedisBloom模块) ：
   >
   >      ```
   >      BF.RESERVE myBloomFilter 0.01 1000000
   >      ```
   >
   >      上述命令创建了一个误判率为1%且容量为100万的布隆过滤器。
   >
   >   2. 添加元素：
   >
   >      ```
   >      BF.ADD myBloomFilter "item1"
   >      ```
   >
   >   3. 检查元素是否存在：
   >
   >      ```
   >      BF.EXISTS myBloomFilter "item1"    # 返回 1（可能存在）
   >      BF.EXISTS myBloomFilter "item2"    # 返回 0（一定不存在）
   >      ```
   >
   > **适用场景**
   > 布隆过滤器一般都在海量数据判断场景，可以允许误判：
   >
   > 1. 爬虫：
   >    对已经爬取过的海量URL去重。
   > 2. 黑名单：
   >    例如反垃圾邮件，用于判断一个邮件地址是否在黑名单中，提高垃圾邮件过滤的效率(可能会误杀)。
   > 3. 分布式系统：
   >    用于判断数据是否在某个节点上，减少网络请求,提高系统性能。例如，Hadoop 和Cssandra都使用布隆过滤器来优化数
   >    据分布和查找。
   > 4. 推荐系统：
   >    用于判断用户否已经看过某个推荐内容，避免重复推荐。

   

## 2. 缓存击穿

### 2.1 介绍

![redis_oscnan](E:\各种资料\Java开发笔记\我的笔记\images\redis_oscnan.png)

缓存击穿指的是某一热点数据缓存失效，使得大量请求直接打到了数据库，增加数据负载。

想象一下大家都在抢茅台，但在某一时刻茅台的缓存失效了，大家的请求打到了数据库中，这就是缓存击穿。

缓存雪崩是多个key，大量数据同时过期，而缓存击穿是某个热点key崩溃，可以认为缓存击穿是缓存雪崩的子集。

### 2.2 解决方法

1. 加互斥锁：保证同一时间只有一个请求来构建缓存，跟缓存雪崩相同。
2. 热点数据永不过期：不要给热点数据设置过期时间，在后台异步更新缓存。



## 3. 缓存雪崩

### 3.1 介绍

![redis_cosans](E:\各种资料\Java开发笔记\我的笔记\images\redis_cosans.png)

缓存雪崩是指在某个时间点，大量缓存同时失效或被清空，导致大量请求直接打到数据库或后端系统，造成系统负载激增，甚至引发系统崩溃。这通常是由于缓存中的大量数据在同一时间失效引起的。

想象一个电商系统，用户量很大，假设某一系列的商品缓存突然同一时间失效，那就会造成我们的缓存雪崩，导致服务全部打到了数据库,引发严重后果。

### 3.2 解决办法
缓存键同时失效:

1. 过期时间随机化：设置缓存的过期时间，加上一个随机值,避免同-时间大量缓存失效。
2. 使用多级缓存：引入多级缓存机制，如本地缓存和分布式缓存相结合，减少单点故障风险。
3. 缓存预热：系统启动时提前加载缓存数据,避免大量请求落到冷启动状态下的数据库。
4. 加互斥锁：使得没缓存或缓存失效的情况下，同一时间只有一个请求来构建缓存，防止数据库压力过大。

缓存中间件故障:

1. 服务熔断：暂停业务的返回数据,直接返回错误。
2. 构建集群：构建多个Redis群保证其可用。



# 九、Redis实战案例

## 1. 排行榜

## 2. 

