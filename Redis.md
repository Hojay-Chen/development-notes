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

- int编码：如果一个字符串可以被解析为证书，并且整数值比较小，Redis会直接使用整数编码。

  ```c
  struct redisObject {
      unsigned type:4;      // 数据类型（字符串、哈希等）
      unsigned encoding:4;  // 编码类型（int、embstr、raw等）
      int64_t ptr;          // 实际的数据指针，这里直接存储整数值
  };
  ```

  比如直接存储123，则：

  ![redis_string_nvodabo](E:\各种资料\Java开发笔记\我的笔记\images\redis_string_nvodabo.png)

- embstr编码：当字符串长度比较短（小于等于44字节），Redis会使用embstr编码，这种编码将所有的字符串相关结构体和字符串数据存放在连续的内存块中，分配内存的时候，只需要分配一次，减少内存分配和管理的开销。

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

  ![redis_string_onqbf](E:\各种资料\Java开发笔记\我的笔记\images\redis_string_onqbf.png)

- row编码：当字符串长度超过44字节时，Redis会使用raw编码，这种编码方式将结构体和实际字符串数据分开存储，以便处理更长的数据。

  > redis4.0版本及之后的版本，这个界限是44，前面版本是39。

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

  ![redis_string_qpnicna](E:\各种资料\Java开发笔记\我的笔记\images\redis_string_qpnicna.png)

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

- Redis6及之前，Hash数据结构底层是通过hashtable + ziplist来实现的。

- Redis7之后，Hash数据结构底层是通过hashtable + Listpack来实现的。

ziplist和listpack查找key的效率是类似的，时间复杂度都是0 (n) ，其主要区别就在于listpack 解决了ziplist 的级联更新问题。
Redis内有两个值，分别是hash-max-ziplist-entries ( hash-max-listpack-entries )以及hash-max- ziplist-value ( hash-max-listpack-value )，即Hash类型键的字段个数(默认512) 以及每个字段名和字段值的长度(默认64)。这两个值是可以修改的，通过如下指令进行修改：

```
config set hash-max- zip L ist-entries 4399  # 更改为4399
config set hash-max - z iplist -value 2024  # 更改为2024
```

> redis 7.0为了兼容早期的版本，没有把ziplist相关的值删掉。

- 当hash小于这两个值的时候，会使用listpack或者ziplist 进行存储。
- 当大于这两个值的时候会使用hashtable 进行存储。

这里需要注意一个点， 在使用hashtable结构之后，就不会再退化成ziplist或listpack，之后都是使用hashtable进行存储。

> #### ziplist实现原理
>
> Ziplist (压缩列表)是一种紧凑的数据结构，它将所有元素紧密排列存储在单个连续内存块中，十分节省空间。
>
> 这些元素可以是字符串或整数，且每个元素的存储都紧凑地排列在一起。
>
> 以下是ziplist的具体结构:
>
> ![redis_ziplist_obgabs](E:\各种资料\Java开发笔记\我的笔记\images\redis_ziplist_obgabs.png)
>
> - zlbytes: 记录整个ziplist所占用的字节数。
> - zltail: 记录ziplit中最后一个节点距离ziplist起始地址的偏移量。
> - entry:各个节点的数据。
> - zllen: 记录ziplist中节点的个数。
> - zlend: 特殊值0xFF,用于标记ziplist的结束。
>
> entry的结构如下,会记录前一个节点的长度和编码：
>
> ![redis_ziplist_nonfas](E:\各种资料\Java开发笔记\我的笔记\images\redis_ziplist_nonfas.png)
>
> 因为entry需要记录前一个元素的大小，如果前面插入的元素很大,则已经存在的entry的pre_ entry_length毀需要变大，改变大后续的节点也需要变，所以可能导致级联更新的情况，影响性能。
> 查询需按顺序遍历所有元素,逐个检查是否匹配查询条件。
>
> 
>
> #### Listpack实现原理
>
> Listpack采用一种紧凑的格式来存储多个元素(本质上仍是一个字节数组) ，并组使用多种编码方式来表示不同长度的数据。
>
> Listpack的结构如下：
>
> ![redis_listpack_zbnojabs](E:\各种资料\Java开发笔记\我的笔记\images\redis_listpack_zbnojabs.png)
>
> - header: 整个listpack的元数据，包括总长度和总元素个数。
> - elements: 实际存储的元素，每个元素包括长度和数据部分。
> - end: 标识listpack结束的特殊字节。
>
> element的内部结构如下：
>
> ![redis_listpack_zanofdna](E:\各种资料\Java开发笔记\我的笔记\images\redis_listpack_zanofdna.png)
>
> - encoding-type: 元素的编码类型。
> - element-data: 实际存放的数据。
> - element-tot-len: encoding-type + element-data的总长度，包含自己的长度。
>
> 之所以设计它来替换ziplist就是因为ziplist连锁更新的问题，因为ziplist的每个entry会记录之前的entry长度。
>
> 如果前面的entry长度变大，那么当前entry记录前面entry的字段所需的空间也需要扩大,而当前的大了，可能后面的entry也得大,这就是所谓的连锁更新，比较影响性能。
>
> 而ListPack的每个元素，仅记录自己的长度，这样一来修改会新增不会影响后面的长度变大，也就避免了连锁更新的问题。
>
> 
>
> #### hashtable实现原理
>
> Hashtable就是哈希表实现，查询时间复杂度为0(1)，效率非常快。HashTable的结构如下：
>
> ```c
> typedef struct dictht {
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
> - size: 示哈希表的大小。
> - sizemask: 这个是指哈希表大小的掩码，它的值永远等于size-1, 这个属性和哈希值一起约定了哈希节点所处的哈希表的位置,引的值index = hash (哈希值) & sizemask。
> - used: 示已经使用的节点数量。
>
> ![redis_hashtable_aojdq](E:\各种资料\Java开发笔记\我的笔记\images\redis_hashtable_aojdq.png)
>
> 我们再看下哈希节点(dictEntry) 的组成，它主要由三个部分组成,分别为key，value 以及指向下一个哈希节点的指针，其源码中结构体的定义如下：
>
> ```c
> typedef struct dictEntry {
>     //键值对中的键
>     void *key;
>   
>     //键值对中的值
>     union {
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
> ![redis_hashtable_xaosbno](E:\各种资料\Java开发笔记\我的笔记\images\redis_hashtable_xaosbno.png)
>
> **渐进式rehash**
> 在平时，插入数据的时候，所有的数据都会写入ht[0]即哈希表1, ht [1]哈希表2此时就是一-张没有分配空间的空表。
> 但是随着数据越来越多，当dict的空间不够的时候，就会触发扩容条件,扩容流程主要分为三步:
>
> 1. 首先，为哈希表2即分配空间。新表的大小是第一 个大于等于原表2倍used的2次方幂。举个例子，如果原表即哈希表1的值是1024, 那个其扩容之后的新表大小就是2048。分配好空间之后，此时dict就有了两个哈希表了，然后此时字典的rehashidx即rehash索引的值从-1暂时变成0 ，然后便开始数据转移操作。
> 2. 数据开始实现转移。每次对hash进行增删改查操作，都会将当前rehashidx 的数据从在哈希表1迁移到2上，然后rehashidx+1，所以迁移的过程是分多次、渐进式地完成。
>    注意:插入数据会直接插入到2表中。
> 3. 随着操作不断执行，最终哈希表1的数据都会被迁移到2中,这时候进行指针对象进行互换，即哈希表2变成新的哈希表1,而原先的哈希表1成哈希表2并且设置为空表，最后将rehashidx的值设置为-1。
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

Redis中的List数据结构底层是通过压缩列表（ZipList）或双向链表（LinkedList）来实现的。当List中的元素较少时，使用压缩列表存储，节省内存；当元素较多时，自动转换为双向链表。

[[Redis 的 list 底层实现原理详解](https://www.mianshiya.com/bank/1791375592078610434/question/1799677102985125890#heading-3)]

**优势**：

- **高效的数据存储**：可以存储多个元素，节省内存。
- **快速插入和删除**：在List的头部或尾部插入和删除元素的时间复杂度为O(1)。
- **灵活性**：可以动态添加、删除和修改元素。

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



## 4. Set

### 4.1 底层实现原理

Redis中的Set数据结构底层是通过哈希表或整数集合（IntSet）来实现的。当集合中的所有元素都是整数且元素数量较少时，使用整数集合（IntSet）存储，节省内存；当元素较多或包含非整数元素时，自动转换为哈希表。

**优势**：

- **高效的数据存储**：可以存储多个唯一元素，节省内存。
- **快速查找**：通过哈希表实现，查找元素的时间复杂度为O(1)。
- **灵活性**：可以动态添加、删除和修改元素。
- **集合操作**：支持交集、并集、差集等集合操作。

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



## 5. ZSet（Sorted Set）

### 5.1 底层实现原理

Redis中的ZSet（有序集合）数据结构底层是通过跳跃表（Skip List）和哈希表来实现的。每个ZSet键对应一个跳跃表和一个哈希表，跳跃表用于维护元素的有序性，哈希表用于快速查找元素。

[[Redis 的 list 底层实现原理详解](https://www.mianshiya.com/bank/1791375592078610434/question/1861688534670434305)]

[[Redis 中跳表底层实现原理详解](https://www.mianshiya.com/bank/1791375592078610434/question/1780933295597449217)]

**优势**：

- **高效的数据存储**：可以存储多个成员及其分数，节省内存。
- **快速查找**：通过哈希表实现，查找成员的时间复杂度为O(1)。
- **有序性**：通过跳跃表维护成员的有序性，支持范围查询。
- **灵活性**：可以动态添加、删除和修改成员及其分数。

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



# 四、发布订阅模式

## 1. 介绍
Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。

Redis 客户端可以订阅任意数量的频道。

## 2. Redis的发布和订阅
客户端可以订阅频道，如下图：

![redis_pubsub_xnonda](E:\各种资料\Java开发笔记\我的笔记\images\redis_pubsub_xnonda.png)

当给这个频道发布消息后，消息就会发送给订阅的客户端，如下图：

![redis_pubsub_xnoas](E:\各种资料\Java开发笔记\我的笔记\images\redis_pubsub_xnoas.png)

## 3. 发布订阅命令行实现
（1）打开一个客户端订阅channel1
```
SUBSCRIBE channel1
```

![redis_pubsub_pajsjf](E:\各种资料\Java开发笔记\我的笔记\images\redis_pubsub_pajsjf.png)

（2）打开另一个客户端，给channel1发布消息hello
```
publish channel1 hello
```

![redis_pubsub_nvhsjs](E:\各种资料\Java开发笔记\我的笔记\images\redis_pubsub_nvhsjs.png)

返回的1是订阅者数量

（3）打开第一个客户端可以看到发送的消息

![redis_pubsub_njdkd](E:\各种资料\Java开发笔记\我的笔记\images\redis_pubsub_njdkd.png)



# 五、Redis持久化

Redis提供两种主要的持久化机制:

- RDB (Redis Database)快照: 

  - RDB通过**生成某一时刻的数据快照**来实现持久化的，可以在特定时间间隔内保存数据的快照。

  - 适给灾难恢复和备份，能生成紧凑的二进制文件，但可能会在崩溃时丢失最后一次快照之 后的数据。

- AOF (Append Only File)日志:

  - AOF 通过将每个写操作追加到日志文件中实现持久化，支持将所有写操作记录下来以便恢复。

  - 数据恢复更为精确，文件体积较大，写时可能会消耗更多资源。

- Redis 4.0新增了RDB和AOF的混合持久化机制。

## 1. RDB

### 1.1 概述

RDB持久化通过创建快照来获取内存某个时间点上的副本，利用快照可以进行方便地进行主从复制，默认快照文件为dump.rdb。

redis.conf文件可以配置在x秒内如果至少有y个key发生变化就会触发命令进行持久化操作。

### 1.2 过程

#### 1.2.1 save

由主进程生成RDB文件，因此生成其间，主进程**无法执行正常的读写命令**，需要等待RDB结束。

#### 1.2.2 bgsave

![redis_RDB_noknas](E:\各种资料\Java开发笔记\我的笔记\images\redis_RDB_noknas.png)

1. **检查子进程是否存在AOF/RDB的子进程正在进行**，如果有返回错误。

2. **父进程触发RDB持久化**：

   - 当Redis配置为RDB持久化模式时，父进程会定期触发RDB持久化操作。这可以通过配置文件中的`save`指令来设置，例如`save 900 1`表示如果900秒内至少有1个键被修改，则触发RDB持久化。

   - 也可以通过命令`BGSAVE`手动触发RDB持久化。

3. **创建子进程**：

   - 父进程通过`fork()`系统调用创建一个子进程，在创建子进程期间父进程是阻塞的，无法响应指令的。在`fork()`调用时，子进程会复制父进程的内存空间，但由于操作系统的写时复制（COW）机制，此时并不会真正复制内存内容，而是共享相同的物理内存页，如下图：

     ![redis_RDB_cnoan](E:\各种资料\Java开发笔记\我的笔记\images\redis_RDB_cnoan.png)

   - 只有当父进程或子进程对内存进行写操作时，才会触发COW机制，真正复制内存页，如下图：

     ![redis_RDB_csnka](E:\各种资料\Java开发笔记\我的笔记\images\redis_RDB_csnka.png)

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

- Fork 操作会产生短暂的阻塞,微秒级别操作过后，不会阻塞主进程，整个过程不是完全的非阻塞。
- RDB于是快照备份所有数据，而不是像AOF -样存写命令,因为Redis实例重启后恢复数据的速度可以得到保证,大数据量下比AOF会快很多。
- Fork 操作利用了写时复制，类似与CopyOnWriteArrayList。
  

## 2. AOF



# 六、Redis集群

## 1. 原理

### 1.1 集群信息同步

Redis集群内每个节点都会保存集群的完整拓扑信息，包括每个节点的ID、IP 地址、端口、负责的哈希槽范围等。节点之间使用Gossip协议进行状态交换，以保持集群的一致性和故障检测。每个节点会周期性地发送PING和PONG消息，交换集群信息，使得集群信息得以同步。

> **Gossip的优点**
>
> - 快速收敛: Gossip协议能够快速传播信媳，确保集群状态的迅速更新。
>
> - 降低网络负担:于信息是以随机节点间的对话访式传播，避免了集中式的状态查询，从而降低了网络流量。

### 1.2 集群分片

Redis集群会将数据分散到16384 (2 ^ 14)个哈希槽中，集群中的每个节负责-定范围的哈希槽，在Redis集群中，使用CRC16哈希算法计算键的哈希槽，以确定该键应存储在哪个节点。
集群哈希槽分片如图所标：

<img src="E:\各种资料\Java开发笔记\我的笔记\images\redis_Sacnjkn.png" alt="redis_Sacnjkn" style="zoom:67%;" />

每个节点会拥有一部分的槽位， 然后对应的键值会根据其本身的key，映射到一个哈希槽中，其主要流程如下:

- 根据键值的key，按照CRC 16算法计算一个16 bit的值，然后将16 bit的值对16384进行取余运算，后得到一个对应的哈希槽编号。
- 根据每个节纷配的哈希槽区间，对应编号的数据落在对应的区间上，就能找到对应的分实例。

**注意**：redis 客户端可以访问集群中任意一台实例， 正常情况下这个实例包含这个数据。但如果槽被转移了，客户端还未来得及更新槽的信息，当前实例没有这个数据，则返回MOVED响应给客户端，将其重定向到对应的实例(Gossip集群内每个节点都会保存集群的完整拓扑信息)

> ## 为什么是16384个槽呢？
>
> （1）首先是消息大小的考虑。
> 正常的心跳包需要带上节点完整配置数据，心跳还是比较频繁的，所以需要考虑数据包的大小，如果使用16384数据包只要2k，如果用了65535则需要8k。
> 实际上槽位信息使用一个帐度为16384位的数组来表示，节点拥有哪个槽位，就将对应位置的数据信息设置为1，否则为0.
> 心跳数据包就包含槽位信息如下图所示:
>
> ![redis_mnvlan](E:\各种资料\Java开发笔记\我的笔记\images\redis_mnvlan.png)
>
> 这里我们看到一个重点，即在消息头中最占空间的是myslots[CLUSTER_ SLOTS/8]。
>
> - 当槽位为65536时，这块的大小是: 65536+8+ 1024=8kb
> - 当槽位为16384时， 这块的大小是: 16384+8+ 1024=2kb
>
> 如果槽位为65536，这个ping消息的消息头就太大了，浪费带宽。
>
> （2）集群规模的考虑。
>
> 集群不太可能会扩展超过1000个节点，16384 够用且使得每个分片下的槽又不会太少。



Redis 事务
介绍
https://redis.io/docs/manual/transactions/

可以一次执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，按顺序地串行化执行而不会被其它命令插入，不许加塞

作用
一个队列中，一次性、顺序性、排他性的执行一系列命令

Redis事务与数据库事务的区别
1 、单独的隔离操作

Redis的事务仅仅是保证事务里的操作会被连续独占的执行，redis命令执行是单线程架构，在执行完事务内所有指令前是不可能再去同时执行其他客户端的请求的

2 、没有隔离级别的概念

因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这种问题了

3、不保证原子性

Redis的事务不保证原子性，也就是不保证所有指令同时成功或同时失败，只有决定是否开始执行全部指令的能力，没有执行到一半进行回滚的能力

4、 排它性

Redis会保证一个事务内的命令依次执行，而不会被其它命令插入

使用
常用命令
下表列出了 redis 事务的相关命令：

序号	命令及描述
1	DISCARD 取消事务，放弃执行事务块内的所有命令。
2	EXEC 执行所有事务块内的命令。
3	MULTI 标记一个事务块的开始。
4	UNWATCH 取消 WATCH 命令对所有 key 的监视。
5	[WATCH key key …] 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
不用场景说明
场景一：正常执行


场景二：放弃事务


场景三：全体连坐


场景四：冤头债主
非编译错误，在执行时出错，其他命令不受影响



场景五：watch监控
Redis使用Watch来提供乐观锁定，类以于CAS(Check-and-Set）

CAS回顾：

1、悲观锁

悲观锁(Pessimistic Lock)， 顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。

2、乐观锁

乐观锁(Optimistic Lock)， 顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据。

乐观锁策略:提交版本必须 大于 记录当前版本才能执行更新

3、CAS



Watch:



小结：

一旦执行了exec之前加的监控锁都会被取消

当客户端连接丢失的时候（比如退出链接），所有东西都会被取消监视

Redis事务总结
开启：以MUTI开始一个事务

入队：将多个命令入队到事务中，接到这些命令并不会立即执行，而是放到等待执行的事务队列里面

执行：由EXEC命令触发事务

Redis管道（pipeline）
开篇先来个面试题

如何优化频繁命令往返造成的性能瓶颈？
Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。一个请求会遵循以下步骤：

1 、客户端向服务端发送命令分四步(发送命令→命令排队→命令执行→返回结果)，并监听Socket返回，通常以阻塞模式等待服务端响应。

2、服务端处理命令，并将结果返回给客户端。

上述两步称为：Round Trip Time(简称RTT，数据包往返于两端的时间)



如果同时需要执行大量的命令，那么就要等待上一条命令应答后再执行，这中间不仅仅多了RTT（Round Time Trip），而且还频繁调用系统IO，发送网络请求，同时需要Redis调用多次read()和write()系统方法，系统方法会将数据从用户态转移到内核态，这样就会对进程上下文有比较大的影响了，性能不太好

对于上述问题的解决思路就是采用管道(pipeline)

Redis管道介绍
官网介绍： https://redis.io/docs/manual/pipelining/

Pipeline是为了解决RTT往返回时，仅仅是将命令打包一次性发送，对整个Redis的执行不造成其它任何影响

是批做处理命令变种优化措施，类似Redis的原生批命令(mget和mset)

案例演示
可以将所有要批量执行的命令写入到一个文件中，然后使用 --pipe 命令参数执行



Redis管道小总结
Pipeline与原生批量命令对比
原生批量命令是原子性（例如：mset，mget)，pipeline是非原子性
原生批量命令一次只能执行一种命令，pipeline支持批量执行不同命令
原生批命令是服务端实现，而pipeline需要服务端与客户端共同完成
Pipeline与Redis事务对比
事务具有原子性(不保证)，管道不具有原子性
管道一次性将多条命令发送到服务器，事务是一条一条的发，事务只有在接收到exec命令后才会执行，管道不会
执行事务时会阻塞其他命令的执行，而执行管道中的命令时不会
使用Pipeline注意事项
pipelines缓冲的指令只是会依次执行，不保证原子性，如果执行中指令发生异常，将会继续执行后续的指令
使用pipeline组装的命令个数不能太多，不然数据量过大客户端阻塞的时间可能过久，同时服务端此时也被迫回复一个队列答复，占用很多内存。
Redis发布订阅
专业的事情交给专业的中间件处理，了解即可

介绍
官网介绍： https://redis.io/docs/manual/pubsub/

是一种消息通信模式：发送者(PUBLISH)发送消息，订阅者(SUBSCRIBE)接收消息，可以实现进程间的消息传递

Redis可以实现消息中间件MQ的功能，通过发布订阅实现消息的引导和分流。

作用
Redis客户端可以订阅任意数量的频道，类似我们微信关注多个公众号



当有新消息通过PUBLISH命令发送给频道channel1时



常用命令
下表列出了 redis 发布订阅常用命令：

序号	命令及描述
1	[PSUBSCRIBE pattern pattern …] 订阅一个或多个符合给定模式的频道。
2	[PUBSUB subcommand argument [argument …]] 查看订阅与发布系统状态。
3	PUBLISH channel message 将信息发送到指定的频道。
4	[PUNSUBSCRIBE pattern [pattern …]] 退订所有给定模式的频道。
5	[SUBSCRIBE channel channel …] 订阅给定的一个或多个频道的信息。
6	[UNSUBSCRIBE channel [channel …]] 指退订给定的频道。
推荐先执行订阅后再发布，订阅成功之前发布的消息是收不到的

缺点
发布的消息在Rds系统中不能持久化，因此，必须先执行订阅，再等待消息发布。如果先发布了消息，那么该消息由于没有订阅者，消息将被直接丢弃。
消息只管发送对于发布者而言消息是即发即失的，不管接收，也没有ACK机制，无法保证消息的消费成功
以上的缺点导致Redis的Pub/Sub模式就像个小玩具，在生产环境中几乎无用武之地，为此Redis5.0版本新增了Stream数据结构，不但支持多播，还支持数据持久化，相比Pub/Sub更加的强大
Redis复制（Replication）
介绍
官网介绍： https://redis.io/docs/management/replication/

就是主从复制，master以写为主，Slave以读为主

当master数据变化的时候，自动将新的数据异步同步到其它slave数据库

作用
读写分离
容灾恢复
数据备份
水平扩容支撑高并发
使用
注意点
配从(库)不配主(库)
Master如果配置了requirepass参数，需要密码登陆，那么Slave就要配置masterauth来设置校验密码，
否则Master会拒绝Slave的访问请求
基本命令
1、info replication：

可以查看复制节点的主从关系和配置信息

2、replicaof 主库IP 主库端口:

设置从节点的主节点

一般写入进redis.conf配置文件内

3、slaveof 主库IP 主库端口:

每次与master断开之后，都需要重新连接，除非你配置进redis.conf文件

在运行期间修改slave节点的信息，如果该数据库已经是某个主数据库的从数据库，那么会停上和原主数据库的同步关系转而和新的主数据库同步，重新拜码头

4、slaveof no one：

使当前数据库停止与其他数据库的同步，转成主数据库，自立为王

案例演示
架构说明
个Master两个Slave，需要3台虚机，每台都安装redis



操作步骤
1、三边网络相互ping通且注意防火墙配置

2、拷贝多个redis.conf文件

# 192.168.10.66  redis_6379.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6379
logfile "/my-redis7/redis6379.log"
pidfile /my-redis7/redis6379.pid
dir /my-redis7
dbfilename dump6379.rdb
appendonly yes
appendfilename "appendonly6379.aof"
requirepass shiguang
masterauth shiguang

# 192.168.10.88  redis_6378.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6378
logfile "/my-redis7/redis6378.log"
pidfile /my-redis7/redis6378.pid
dir /my-redis7
dbfilename dump6378.rdb
appendonly yes
appendfilename "appendonly6378.aof"
requirepass shiguang
replicaof 192.168.10.66 6379
masterauth shiguang

# 192.168.10.99  redis_6380.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6380
logfile "/my-redis7/redis6380.log"
pidfile /my-redis7/redis6380.pid
dir /my-redis7
dbfilename dump6380.rdb
appendonly yes
appendfilename "appendonly6380.aof"
requirepass shiguang
replicaof 192.168.10.66 6379
masterauth shiguang
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
修改配置文件细节
1、开启后台启动daemonize yes



2、注释绑定IP bind 127.0.0.1



3、关闭保护模式 protected-mode no



4、指定端口 port <port>



5、指定当前工作目录 dir <path>



6、pid文件名字，pidfile <path>



7、日志文件名称 logfile <path>



8、本机访问密码 requirepass <password>



9、rdb文件名称 dbfilename <filename>



10、aof文件名称 appendfilename <filename>

需启用aof: appendonly yes



可选步骤，默认为 appendonly.aof



11、从机访问主机的通行密码masterauth，从机必须配置



三大命令
1、主从复制：

replicaof 主库IP 主库端口

配从（库）不配主（库）

2、该换门庭：

slaveof 新主库IP 新主库端

3、自立为王：

slaveof no one

一主二仆
方案一：配置文件固定写死 replicaof 主机IP 主机端口

配从不配主，分别配置从机 192.168.10.88 :redis_6378.conf，192.168.10.99: redis_6380.conf

# 192.168.10.88  redis_6378.conf
replicaof 192.168.10.66 6379
masterauth shiguang

# 192.168.10.99  redis_6380.conf
replicaof 192.168.10.66 6379
masterauth shiguang
1
2
3
4
5
6
7
先启动Master，后启动Slaver

主从关系查看：

1、日志

主机日志



从机日志



2、命令 info replication

主机



从机



主从问题

1、从机可以执行写命令吗？

答：不可以



2、从机切入点问题

slave是从头开始复制还是从切入点开始复制?

master启动，写到k3

slave1跟着master同时启动，跟着写到k3

slave2写到k3后才启动，那之前的是否也可以复制？

答： 是，首次一锅端，后续跟随，master写，slave跟

3、主机shutdown后，从机会上位吗？

答：不会，从机不动，原地待命，从机数据可以正常使用；等待主机重启动归来



4、主机shutdown，重启后主从关系还在吗？从机还能否顺利复制？

答： 主从关系还在，从机仍可以顺利复制

5、某台从机shutdown后，master继续写入，从机重启后还能读取新写入的内容吗？

答：可以，同问题2

方案二：命令操作手动指定

从机停机去掉配置文件中的配置项，让3台都是主机状态，各不从属

使用 slaveof 主库IP 主库端口进行指定



使用命令指定Master，从机重启后主从关系还在吗？

答： 不再了，使用命令指定是临时性的。

新货相传
上一个slave可以是下一个slave的master，slavel同样可以接收其他slaves的连接和同步请求

slave成为其他slave的master，本质仍然是slave，不能执行写操作

中途变更转向：会清除之前的数据，重新建立拷贝最新的

反客为主
使当前数据库停止与其他数据库的同步，转成主数据库

命令： slaveof no one



复制原理和工作流程
1、slave启动，同步初请

slave启动成功连接到master后会发送一个sync命令

slave首次全新连接master，一次完全同步（全量复制）将被自动执行，slave自身原有数据会被master数据覆盖清除

2、首次连接，全量复制

naster节点收到sync命令后会开始在后台保存快照（即RDB特久化，主从复制时会触发RDB)，同时收集所有接收到的用于修改数据集命令缓存起来，master节点执行RDB持久化完后，master将rdb快照文件和所有缓存的命令发送到所有slave,以完成一次完全同步

3、心跳持续，保持通信

可在配置文件修改ping包的周期，repl-ping-replica-period <second>

master发出PING包的周期，默认是10秒



4、进入平稳，增量复制

Master继续将新的所有收集到的修改命令自动依次传给slave,完成同步

5、从机下线，重连续传

master会检查backlog里面的offset,master和slave都会保存一个复制的offseti还有一个masterId,offset是保存在backlog中的。Master只会把已经复制的offset后面的数据复制给Slave,类似断点续传

复制缺点
1、复制延时，信号衰减

由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。



2、Masetr宕机后从机不能上位为新的Master

默认情况下，不会在slave节点中自动重选一个master，需要进行人工干预。

Redis哨兵（Sentinel）
介绍
吹哨人巡查监控后台master主机是否故障，如果故障了根据投票数自动将某一个从库转换为新主库，继续对外服务

作用
俗称: 无人值守运维

1、监控Redis运行状态，包括master和slave

2、当master down机，能自动将slave切换成新master



主要功能
1、主从监控

监控主从Redis库运行是否正常

2、故障转移
如果Master异常，则会进行主从切换，将其中一个Slave作为新Master

3、消息通知
哨兵可以将故障转移的结果发送给客户端

4、配置中心

客户端通过连接哨兵来获得当前Redis服务的主节点地址

实战演练
架构说明
3个哨兵 + 一主二从



哨兵负责监控和维护集群，不存放数据，只是吹哨人

一主二从用于数据读取和存放

案例步骤
1、/my-redis7 目录下新建或拷贝sentinel.conf文件，文件名称不可改



2、修改哨兵配置文件

可将哨兵配置到不同机器，此处全配置到192.168.10.66

配置文件如下

# sentinel26379.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 26379
logfile "/my-redis7/sentinel26379.log"
pidfile /var/run/redis-sentinel26379.pid
dir /my-redis7
sentinel monitor mymaster 192.168.10.66 6379 2
sentinel auth-pass mymaster shiguang

# sentinel26380.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 26380
logfile "/my-redis7/sentinel26380.log"
pidfile /var/run/redis-sentinel26380.pid
dir /my-redis7
sentinel monitor mymaster 192.168.10.66 6379 2
sentinel auth-pass mymaster shiguang


# sentinel26381.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 26381
logfile "/my-redis7/sentinel26381.log"
pidfile /var/run/redis-sentinel26381.pid
dir /my-redis7
sentinel monitor mymaster 192.168.10.66 6379 2
sentinel auth-pass mymaster shiguang
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
3、配置一主二从关系

66机器上新建redis6379.conf配置文件并设置masterauth项访问密码为shiguang，不然后续可能报错master_link_status:down，因为Master下线变为从机后需要访问新的Master

88机器上新建redis_6378.conf配置文件，设置好replicaof <masterip> <masterport>

99机器上新建redis_6380.conf配置文件，设置好replicaof <masterip> <masterport>

4、分别启动三台服务并测试主从复制是否正常



5、分别启动三个哨兵

redis-sentinel /path/to/sentinel.conf

或者

redis-server /path/to/sentinel.conf --sentinel

redis-sentinel /my-redis7/sentinel26379.conf 
redis-sentinel /my-redis7/sentinel26380.conf 
redis-sentinel /my-redis7/sentinel26381.conf
1
2
3
Master 主机服务如下，一台Redis服务，三个哨兵服务


查看其中一个哨兵日志



6、手动shutdown将6379下线，模拟master宕机

思考几个问题

1、两台从机数据是否正常?

2、是否会从剩下两台机器中选出新的master?

3、之前的master重新恢复后，谁是老大，会不会master冲突？

答案揭晓

1、两台从机数据正常，但是可能会出现下面两个问题



认识broken pipe



pipe是管道的意思，管道里面是数据流，通常是从文件或网络套接字读取的数据。当该管道从另一端突然关闭时，会发生数据突然中断，即是broken，对于socket来说，可能是网络被拔出或另一端的进程崩溃

解决问题

当该异常产生的时候，对于服务端来说，并没有多少影响。因为可能是某个客户端突然中止了进程导致了该错误

Broken Pipe总结

这个异常是客户端读取超时关闭了连接,这时候服务器端再向客户端已经断开的连接写数据时就发生了broken pipe异常！

2、会重新投票选出新的Master




3、之前的Master恢复后继续延用新选出的Master



sentinel.conf参数说明
1、bind
服务监听地址，用于客户端连接，默认本机地址
2、daemonize
是否以后台daemon方式运行
3、protected-mode
安全保护模式
4、port
端口
5、logfile
日志文件路径
6、pidfile
pid文件路径
7、dir
工作目录

8、sentinel monitor <master-name> <ip> <redis-port> <quorum>

设置要监控的master服务器

quorum表示最少有几个哨兵认可客观下线，同意故障迁移的法定票数



9、sentinel auth-pass <master-name> <password>

master设置了密码，连接master服务的密码

10、其他

# 指定多少毫秒之后，主节点没有应答哨兵，此时哨兵主观上认为主节点下线

sentinel down-after-milliseconds <master-name> <milliseconds>：

# 表示允许并行同步的slave个数，当Master挂了后，哨兵会选出新的Master，此时，剩余的slave会向新的master发起同步数据
sentinel parallel-syncs <master-name> <nums>：


# 故障转移的超时时间，进行故障转移时，如果超过设置的毫秒，表示故障转移失败


sentinel failover-timeout <master-name> <milliseconds>：

# 配置当某一事件发生时所需要执行的脚本
sentinel notification-script <master-name> <script-path> ：

 # 客户端重新配置主节点参数脚本

sentinel client-reconfig-script <master-name> <script-path>：
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
配置文件变化
配置文件的内容，在运行期间会被sentinel动态进行更改

Master、Slave切换后，Master、Slave的redis.conf、和sentinel.conf的内容都会发生改变

Master redis.conf中会多一行replicaof的配置



sentinel.conf的监控目标会随之调换



其他说明
1、生产都是不同机房不同服务器，很少出现3个哨兵全挂掉的情况

2、可以同时监控多个master,一行一个

哨兵运行流程和选举原理
一般建议sentinel采取奇数台，Sentinel 在选举新的主节点时需要达到法定人数。奇数台 Sentinel 可以更容易地达成多数派，避免出现平局的情况。

1、SDown主观下线（Subjectively Down）
SDOWN(主观不可用)是单个sentinel自己主观上检测到的关于master的状态，从sentinel的角度来看，如果发送了PING心跳后，在一定时间内没有收到合法的回复，就达到了SDOWN的条件。

Sentineli配置文件中的down-after-millisecondsi设置了判断主观下线的时间长度



2、ODown客观下线(Objectively Down)
ODOWN需要一定数量的Sentinel,多个哨兵达成一致意见才能认为一个master客观上已经宕掉

四个参数含义：

masterName是对某个master+slave组合的一个区分标识(一套sentinel可以监听多组master+slave这样的组合)



quorum这个参数是进行客观下线的一个依据，法定人数/法定票数

意思是至少有quorum个sentinel认为这个master有故障才会对这个master进行下线以及故障转移。因为有的时候，某个sentinel节点可能因为自身网络原因导致无法连接master，而此时master并没有出现故障，所以这就需要多个sentinel都一致认为该master有问题，才可以进行下一步操作，这就保证了公平性和高可用。

3、选举出领导者哨兵（哨兵中选出兵王）
当主节点被判断客观下线以后，各个哨兵节点会进行协商，先选举出一个领导者哨兵节点（兵王）并由该领导者节点，也即被选举出的兵王进行failover(故障迁移)







哨兵领导者，兵王如何洗出来的？



监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是Raft算法；Raft算法的基本思路是先到先得：

即在一轮选举中，哨兵A向B发送成为领导者的申请，如果B没有同意过其他哨兵，则会同意A成为领导者

4、由兵王开始推动故障切换流程并选出一个新master
第一步：新主登基
某个Slave被选中成为新Master

选出新master的规则:



1、剩余slave节点健康前提下，redis.conf文件中，优先级slave-priority或者replica-priority最高的从节点（数字越小优先级越高）



2、复制偏移位置offset最大的从节点

3、最小Run Id的从节点（字典顺序，ASCII码）

第二步：群臣俯首
朝天子一朝臣，换个码头重新拜

执行slaveof no one命令让选出来的从节点成为新的主节点，并通过replicaof命令让其他节点成为其从节点

第三步：旧主拜服
老masterl回来也认怂

将之前已下线的老masteri设置为新选出的新master的从节点，当老master重新上线后，它会成为新master的从节点

Sentinel leader会让原来的master降级为slave并恢复正常工作。

小总结
上述的failover操作均由sentinel自己独自完成，完全无需人工干预，

哨兵使用建议
1、哨兵节点的数量应为多个，哨兵本身应该集群，保证高可用

2、哨兵节点的数量应该是奇数

3、个哨兵节点的配置应一致

4、如果哨兵节点部署在Docker等容器里面，尤其要注意端口的正确映射

5、哨兵集群+主从复制，并不保证数据零丢失

Redis集群（Cluster）
介绍
官网介绍：https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/

由于数据量过大，单个Master复制集难以承担，因此需要对多个复制集进行集群，形成水平扩展每个复制集只负责存储整个数据集的一部分，这就是Redis的集群，其作用是提供在多个Redis节点间共享数据的程序集。



Redis集群是一个提供在多个Redis节点间共享数据的程序集，Redis集群可以支持多个Master



作用
1、Redis集群支持多个Master,每个Master又可以挂载多个Slave

读写分离
支持数据的高可用
支持海量数据的读写存储操作
2、由于Cluster自带Sentinel的故障转移机制，内置了高可用的支持，无需再去使用哨兵功能

3、客户端与Rdis的节点连接，不再需要连接集群中所有的节点，只需要任意连接集群中的一个可用节点即可

4、槽位slot负责分配到各个物理服务节点，由对应的集群来负责维护节点、插槽和数据之间的关系

集群算法：分片-槽位（slot）
官网介绍


Slot介绍


分片介绍


插槽和分片的优势


Slot槽位映射方案
方案一：哈希取余分区


方案二：一致性哈希算法分区
一致性Hash算法背景
一致性哈希算法在1997年由麻省理工学院中提出的，设计目标是为了解决分布式缓存数据变动和映射问题，某个机器宕机了，分母数量改变了，自然取余数不OK了。

作用
提出一致性Hash解决方案。目的是当服务器个数发生变动时，尽量减少影响客户端到服务器的映射关系

实现步骤
1、算法构建一致性哈希环



2、Redis服务器IP节点映射



3、Key落到Redis服务器的落键规则



优点
1、容错性方面



2、拓展性方面



缺点
节点太少时，会产生数据倾斜问题



小总结
为了在节点数目发生改变时尽可能少的迁移数据，将所有的存储节点排列在收尾相接的Hash环上，每个key在计算Hash后会顺时针找到临近的存储节点存放。而当有节点加入或退出时仅影响该节点在Hash环上顺时针相邻的后续节点。

优点：

加入和删除节点只影响哈希环中顺时针方向的相邻的节点，对其他节点无影响

缺点 ：

数据的分布和节点的位置有关，因为这些节点不是均匀的分布在哈希环上的，所以数据在进行存储时达不到均匀分布的效果。

方案三：哈希槽分区
产生背景
一致性哈希算法的数据倾斜问题

哈希槽实质就是一个数组，数组[0,2^14 -1]形成hash slot空间。

作用
解决均匀分配的问题，在数据和节点之间又加入了一层，把这层称为哈希槽（slot），用于管理数据和节点之间的关系，现在就相当于节点上放的是槽，槽里放的是数据。



槽解决的是粒度问题，相当于把粒度变大了，这样便于数据移动。哈希解决的是映射问题，使用key的哈希值来计算所在的槽，便于数据分配

多少个hash 槽？
一个集群只能有16384个槽，编号0-16383（0-2^14-1）。这些槽会分配给集群中的所有主节点，分配策略没有要求。

集群会记录节点和槽的对应关系，解决了节点和槽的关系后，接下来就需要对key求哈希值，然后对16384取模，余数是几key就落入对应的槽里。HASH_SLOT = CRC16(key) mod 16384。以槽为单位移动数据，因为槽的数目是固定的，处理起来比较容易，这样数据移动问题就解决了。

为什么Redis集群的最大槽数是16384个？
高频面试题

Redis集群并没有使用一致性hash而是引入了哈希槽的概念。Redis 集群有16384个哈希槽，每个key通过CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分hash槽。但为什么哈希槽的数量是16384（2^14）个呢？

CRC16算法产生的hash值有16bit，该算法可以产生2^16=65536个值。换句话说值是分布在0~65535之间，有更大的65536不用为什么只用16384就够？作者在做mod运算的时候，为什么不mod65536，而选择mod16384？ HASH_SLOT = CRC16(key) mod 65536为什么没启用

作者在GitHub对于该问题的解答：https://github.com/redis/redis/issues/2576



1、说明1

正常的心跳数据包带有节点的完整配置，可以用幂等方式用旧的节点替换旧节点，以便更新旧的配置。

这意味着它们包含原始节点的插槽配置，该节点使用2k的空间和16k的插槽，但是会使用8k的空间（使用65k的插槽）。

同时，由于其他设计折衷，Redis集群不太可能扩展到1000个以上的主节点。

因此16k处于正确的范围内，以确保每个主机具有足够的插槽，最多可容纳1000个矩阵，但数量足够少，可以轻松地将插槽配置作为原始位图传播。请注意，在小型群集中，位图将难以压缩，因为当N较小时，位图将设置的slot / N位占设置位的很大百分比。



2、说明2

(1)如果槽位为65536，发送心跳信息的消息头达8k，发送的心跳包过于庞大。

在消息头中最占空间的是myslots[CLUSTER_SLOTS/8]。 当槽位为65536时，这块的大小是: 65536÷8÷1024=8kb

在消息头中最占空间的是myslots[CLUSTER_SLOTS/8]。 当槽位为16384时，这块的大小是: 16384÷8÷1024=2kb

因为每秒钟，redis节点需要发送一定数量的ping消息作为心跳包，如果槽位为65536，这个ping消息的消息头太大了，浪费带宽。

(2)redis的集群主节点数量基本不可能超过1000个。

集群节点越多，心跳包的消息体内携带的数据越多。如果节点过1000个，也会导致网络拥堵。因此redis作者不建议redis cluster节点数量超过1000个。 那么，对于节点数在1000以内的redis cluster集群，16384个槽位够用了。没有必要拓展到65536个。

(3)槽位越小，节点少的情况下，压缩比高，容易传输

Redis主节点的配置信息中它所负责的哈希槽是通过一张bitmap的形式来保存的，在传输过程中会对bitmap进行压缩，但是如果bitmap的填充率slots / N很高的话(N表示节点数)，bitmap的压缩率就很低。 如果节点数很少，而哈希槽数量很多的话，bitmap的压缩率就很低。

3、计算结论



一个注意点
Redis集群不保证强一致性，这意味着在特定的条件下，Redis集群可能会丢掉一些被系统收到的写入请求命令



集群环境案例步骤
三主三从Redis集群配置
1、需要三台虚拟机，分别创建用于存放Redis集群配置文件的目录

mkdir -p /my-redis7/cluster
1
2、新建六个独立的Redis实例服务，我已为三台虚拟机分别配置固定的静态ip，分别是 66，88，99

# 192.168.10.66:6381  redis_cluster_6381.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6381
logfile "/my-redis7/cluster/cluster6381.log"
pidfile /my-redis7/cluster/cluster6381.pid
dir /my-redis7/cluster
dbfilename dump6381.rdb
appendonly yes
appendfilename "appendonly6381.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6381.conf
cluster-node-timeout 5000

# 192.168.10.66:6382 redis_cluster_6382.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6382
logfile "/my-redis7/cluster/cluster6382.log"
pidfile /my-redis7/cluster/cluster6382.pid
dir /my-redis7/cluster
dbfilename dump6382.rdb
appendonly yes
appendfilename "appendonly6382.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6382.conf
cluster-node-timeout 5000

# 192.168.10.88:6383 redis_cluster_6383.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6383
logfile "/my-redis7/cluster/cluster6383.log"
pidfile /my-redis7/cluster/cluster6383.pid
dir /my-redis7/cluster
dbfilename dump6383.rdb
appendonly yes
appendfilename "appendonly6383.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6383.conf
cluster-node-timeout 5000

# 192.168.10.88:6384 redis_cluster_6384.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6384
logfile "/my-redis7/cluster/cluster6384.log"
pidfile /my-redis7/cluster/cluster6384.pid
dir /my-redis7/cluster
dbfilename dump6384.rdb
appendonly yes
appendfilename "appendonly6384.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6384.conf
cluster-node-timeout 5000

# 192.168.10.99:6385 redis_cluster_6385.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6385
logfile "/my-redis7/cluster/cluster6385.log"
pidfile /my-redis7/cluster/cluster6385.pid
dir /my-redis7/cluster
dbfilename dump6385.rdb
appendonly yes
appendfilename "appendonly6385.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6385.conf
cluster-node-timeout 5000

# 192.168.10.99:6386 redis_cluster_6386.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6386
logfile "/my-redis7/cluster/cluster6386.log"
pidfile /my-redis7/cluster/cluster6386.pid
dir /my-redis7/cluster
dbfilename dump6386.rdb
appendonly yes
appendfilename "appendonly6386.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6386.conf
cluster-node-timeout 5000

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
其中，requirepass 和 masterauth后面跟的是密码

3、分别启动六台Redis实例，以 192.18.10.66 这台虚拟机为例

cd /usr/local/bin
redis-server /my-redis7/cluster/redis_cluster_6381.conf
redis-server /my-redis7/cluster/redis_cluster_6382.conf
1
2
3


4、通过redis-cli命令为六台机器构建集群关系

redis-cli -a shiguang --cluster create --cluster-replicas 1 192.168.10.66:6381 192.168.10.66:6382 192.168.10.88:6383 192.168.10.88:6384 192.168.10.99:6385 192.168.10.99:6386
1
通过 --cluster-replicas 1 选项，Redis 集群会自动为每个主节点分配一个从节点，以实现高可用性和数据冗余。主从节点会尽量分布在不同的物理机器上，以提高容错性。

提示创建三主三从并返回主从关系及主节点槽位信息，询问是否接受，输入yes以继续



M代表Master，S代表slave，可以注意到，6381从节点6384，6383从节点6386，6385从节点6382都在不同的虚拟机

提示[OK]状态代表构建成功



查看并检验集群状态
info replication
info replication 命令用于获取 Redis 实例的复制（replication）信息。这个命令主要用于查看主从复制的状态和配置。

输出内容：

role: 当前实例的角色，可以是 master 或 slave。
connected_slaves: 连接的从节点数量。
master_replid: 主节点的复制 ID。
master_repl_offset: 主节点的复制偏移量。
slave_repl_offset: 从节点的复制偏移量（如果是从节点）。
slave_priority: 从节点的优先级（用于故障转移）。
slave_read_only: 从节点是否只读。
master_host: 主节点的 IP 地址（如果是从节点）。
master_port: 主节点的端口号（如果是从节点）。
master_link_status: 主从连接状态，可以是 up 或 down。
master_last_io_seconds_ago: 最后一次与主节点通信的时间（秒）。
slave_repl_offset: 从节点的复制偏移量。
slave_priority: 从节点的优先级。
slave_read_only: 从节点是否只读。
cluster info
cluster info 命令用于获取 Redis 集群的整体状态信息。这个命令主要用于查看集群的健康状况和性能指标。

输出内容：
cluster_state: 集群的状态，可以是 ok（正常）或 fail（故障）。
cluster_slots_assigned: 已分配的槽位数量。
cluster_slots_ok: 状态正常的槽位数量。
cluster_slots_pfail: 可能失败的槽位数量。
cluster_slots_fail: 失败的槽位数量。
cluster_known_nodes: 集群中已知的节点数量。
cluster_size: 集群的大小（主节点数量）。
cluster_current_epoch: 当前的集群纪元。
cluster_my_epoch: 当前节点的纪元。
cluster_stats_messages_sent: 发送的消息数量。
cluster_stats_messages_received: 接收的消息数量。
cluster nodes
cluster nodes 命令用于获取 Redis 集群中所有节点的详细信息。这个命令主要用于查看集群的拓扑结构和每个节点的状态。

输出内容：
节点 ID: 每个节点的唯一标识符。
IP:端口: 节点的 IP 地址和端口号。
flags: 节点的状态标志，如 master、slave、myself、fail?、fail、handshake 等。
master ID: 如果是从节点，显示其主节点的 ID。
ping-sent: 最后一次发送 ping 的时间戳。
pong-received: 最后一次接收 pong 的时间戳。
config-epoch: 节点的配置纪元。
link-state: 节点的连接状态，如 connected 或 disconnected。
槽位范围: 如果是主节点，显示其负责的槽位范围。
以6381端口的Redis实例为例进行演示

# 指定端口号登录
cd /usr/local/bin
redis-cli -a shiguang -p 6381
1
2
3
使用**info replication**查看单个 Redis 实例的复制状态和配置。



使用**cluster info**查看整个 Redis 集群的状态和性能指标。

仅启用192.168.10.66 这台服务器Redis实例输入的信息如下



启动所有虚拟机实例



使用 cluster nodes 查看集群中所有节点的详细信息和拓扑结构。



三主三从集群读写
对6381新增两个key，查看效果



测试效果只能存入key2，查询也只能查到key2，存入key1时提示要到 6383这台机器上操作

原因:

写入时需要注意槽位的范围区间，需要路由到位



解决措施：

redis-cli -c 是 Redis 命令行界面（CLI）的一个选项，它允许你以“集群模式”连接到 Redis 集群。当你使用 -c 参数启动 redis-cli 时，它会尝试与 Redis 集群交互，并且能够智能地将命令路由到正确的集群节点上。

使用方法

基本的用法是这样的：

redis-cli -c host1 port1 host2 port2 ...
1
这里的 host1 和 port1 是集群中第一个节点的地址，host2 和 port2 是第二个节点的地址，以此类推。你需要提供集群中至少一个节点的信息，但是为了更健壮的连接性，通常提供多个节点的地址是更好的选择。

重要注意事项

自动重定向：当在集群模式下运行时，如果命令需要在不同的槽上执行，redis-cli 会自动处理重定向，并将结果合并为单个输出。
集群拓扑发现：redis-cli 会通过集群的内置机制来发现整个集群的拓扑结构。
数据分布：Redis 集群通过哈希槽（hash slot）来分配数据，共有 16384 个哈希槽，这些哈希槽分布在集群中的各个节点上。redis-cli -c 会根据这个分布情况来决定哪个命令应该发送到哪个节点。
如果你希望连接到一个 Redis 集群，确保你的环境中正确安装了支持集群功能的 redis-cli 版本。此外，如果你的 Redis 集群配置了认证，还需要使用 -a password 参数指定密码。

例如：

redis-cli -c --cluster nodes 127.0.0.1 6379 --cluster-password your_password
1
这里 --cluster nodes 后面跟着的是集群节点的地址列表，而 --cluster-password 则指定了访问集群所需的密码。请注意，实际的命令可能会有所不同，具体取决于你的 Redis 安装和集群的具体配置。

为防止路由失效，可在登录客户端时添加参数 -c，以集群模式连接到Redis集群，c 即 cluster



当我们再次尝试在6381写入key1 时能够正常写入，且写入时提示已自动重定向到对应的6383实例

若要查看集群信息，可用CLUSTER NODES 命令



若要查看某个key对应的槽位值，可以使用 CLUSTER KEYSLOT key 命令



主从容错切换迁移案例
主节点6381和从节点6384切换演示

案例演示
思考两个问题：

1、主节点宕机，从节点能否顺利上位为主节点？

2、宕机的主节点恢复后，能否重新上位，恢复如初？

先手动停止主节点6381实例



从6382实例检查集群服务状态



可以看到，6381状态为fail，6384成功上位变为主节点，集群架构也从原来的三主三从变为三主二从

再启动6381实例，看能否重新上位为主机点



可以看到，即便6381实例重新恢复，依旧是从节点



即便6381恢复，但6384已上位为主节点，6381成为6384的从节点

结论：

1、主节点宕机，从节点能顺利上位为新的主节点

2、宕机的主节点恢复，不能再次恢复到主节点

注意：

集群不保证数据100%一致性，会有一定数据丢失情况

如果某一台主节点宕机了，从节点还没有成功上位，在从节点上位的这个间隙内有新的写入操作，数据将会丢失

手动故障迁移/节点从属调整
由于6384上位为主节点，6381恢复后又不能恢复到主节点，我们想要让6381重新变为主节点该怎么做呢？

我们可以使用 CLUSTER FAILOVER 命令将6381重新调整为主节点

CLUSTER FAILOVER 是 Redis 集群模式下的一个管理命令，用于触发一个手动的故障转移过程。在 Redis 集群中，每个主节点（master）可以有多个从节点（slave），从节点主要用于读取操作或作为备份数据。如果某个主节点出现故障，Redis 集群可以通过自动故障转移机制将从节点提升为主节点，以保持集群的可用性。

命令格式
CLUSTER FAILOVER [NO-RECONFIG]
1
参数说明
NO-RECONFIG：这是一个可选参数，当指定此参数时，不会重新配置从节点为新的主节点。这意味着在故障转移后，旧的从节点将成为新的主节点，但它不会立即创建新的从节点。这对于某些特定的故障转移场景可能有用，比如当管理员想要手动控制后续的从节点设置时。
功能
当你在 Redis 集群中执行 CLUSTER FAILOVER 命令时，将会发生以下事情：

手动触发故障转移：强制将从节点提升为主节点，以替代已知不可用的主节点。
重新分配槽：新提升的主节点将接管原主节点负责的所有哈希槽。
重新配置从节点：如果未指定 NO-RECONFIG 参数，则新的主节点将自动重新配置其从节点关系。
注意事项
执行 CLUSTER FAILOVER 命令通常是在主节点确实已经确认不可用的情况下进行的。
如果集群中有足够的健康节点，并且配置了复制关系，那么自动故障转移通常会在后台默默地完成，不需要人工干预。
在执行 CLUSTER FAILOVER 之前，请确保了解集群的当前状态以及故障转移的影响范围。
示例
假设你想要手动触发一个故障转移过程，可以这样执行命令：

redis-cli -c <cluster-node-1> <cluster-node-2> ... cluster failover
1
或者如果你想手动触发故障转移但不重新配置从节点，可以这样执行：

redis-cli -c <cluster-node-1> <cluster-node-2> ... cluster failover no-reconfig
1
请替换 <cluster-node-1>、<cluster-node-2> 等为你集群中的节点信息。确保你在执行命令前有足够的权限，并且理解该命令对集群的影响。

在6381实例上使用CLUSTER FAILOVER重新配置为主节点



6381重新变为主节点，6384变为6381的从节点

主从扩容案例
1、在192.168.10.99上新建6387，6388两个实例并启动

# 192.168.10.99:6387 redis_cluster_6387.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6387
logfile "/my-redis7/cluster/cluster6387.log"
pidfile /my-redis7/cluster/cluster6387.pid
dir /my-redis7/cluster
dbfilename dump6387.rdb
appendonly yes
appendfilename "appendonly6387.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6387.conf
cluster-node-timeout 5000

# 192.168.10.99:6388 redis_cluster_6388.conf
bind 0.0.0.0
daemonize yes
protected-mode no
port 6388
logfile "/my-redis7/cluster/cluster6388.log"
pidfile /my-redis7/cluster/cluster6388.pid
dir /my-redis7/cluster
dbfilename dump6388.rdb
appendonly yes
appendfilename "appendonly6388.aof"
requirepass shiguang
masterauth shiguang

cluster-enabled yes
cluster-config-file nodes-6388.conf
cluster-node-timeout 5000
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
分别启动实例

cd /usr/local/bin
redis-server /my-redis7/cluster/redis_cluster_6381.conf
redis-server /my-redis7/cluster/redis_cluster_6382.conf
1
2
3
此时新加入的6387，6388节点实例分别是自己的master



2、将新增的6387(空槽号)作为master节点加入到原有集权

redis-cli -a <密码> --cluster add-node <自己实际IP地址>:6387 <自己实际IP地址>:6381
1
6387 就是将要作为master新增节点

6381 就是原来集群节点里面的领路人，相当于6387拜拜6381的码头从而找到组织加入集群

redis-cli -a shiguang --cluster add-node 192.168.10.99:6387 192.168.10.66:6381
1


3、在任意实例检查集群状态

redis-cli -a shiguang --cluster check 192.168.10.66:6381
1


可以看到6387已经加入到集群，但是 0 slots，0 slaves，也就是说暂时还没有分配到槽号

4、重新分配槽号

在任意一个Redis实例使用 RESHARD 命令重新分派槽号

redis-cli -a 密码 --cluster reshard IP地址:端口号
1
例如此处我使用以下命令

redis-cli -a shiguang --cluster reshard 192.168.10.66:6381
1


询问我们想要分配多少个槽号，现在集群中一共有4个master节点，一共16384个槽号，平均分配 16384/4 = 4096

询问我们新分配的槽号谁来接收



复制6387的实例ID



输入all



输入 yes



5、槽位重新分配后重新检查



可以看到6387已经分配了4096个槽号

另外可以看到，新分配的6387的槽号是不连续的，[0-1364],[5461-6826],[10923-12287]，而之前的都是连续的

为什么会出现这个情况呢？原因是重新分配成本太高，所以前3家各自匀出来一部分，从6381/6383/6385三个旧节点分别匀出1364个坑位给新节点6387

6、为6387主节点分配从节点6388

redis-cli -a 密码 --cluster add-node ip:新slave端口 ip:新master端口 --cluster-slave --cluster-master-id 新主节点ID 
1
例如此处我使用的命令为：

redis-cli -a shiguang --cluster add-node 192.168.10.99:6388 192.168.10.99:6387 --cluster-slave --cluster-master-id 54b2de0fa7adb6c014901108af6a2335fe088da8
1


7、重新检查集群状态



可以看到集群已经变为四主四从，且6388的主节点为6387

主从缩容案例
假设我们需要6387和6388下线，恢复三主三从

1、检查集群状况，先获得从节点6388的节点ID

redis-cli -a shiguang --cluster check 192.168.10.99:6388
1


2、从集群中将6388删除

redis-cli -a 密码 --cluster del-node ip:从机端口 从机6388节点ID
1
例如此处我使用命令：

redis-cli -a shiguang --cluster del-node 192.168.10.99:6388 025bd894baf8fd48144165d1d44f16c5b87f9108
1


3、重新检查集群状态



可以看到6387下0 slaves，说明6388已经移除集群，集群中只剩下7个实例

4、将6387槽号清空，将槽号重新分配给6381实例

redis-cli -a shiguang --cluster reshard 192.168.10.66:6381
1


输入 yes



等待分配完成



5、重新检查集群状态

redis-cli -a shiguang --cluster check 192.168.10.66:6381
1


可以看到6387节点变成了6381的从节点

4096个槽位都指给6381，它变成了8192个槽位，相当于全部都给6381了，不然要输入3次，一锅端

6、将6387删除

命令：

redis-cli -a 密码 --cluster del-node ip:端口 6387节点ID
1
例如此处我使用以下命令；

redis-cli -a shiguang --cluster del-node 192.168.10.99:6387 54b2de0fa7adb6c014901108af6a2335fe088da8
1


6、重新检查集群状态



可以看到集群又恢复到了三主三从

集群常用操作命令和CRC16算法分析
通识占位符
不在同一个slot槽位下的多键操作支持不好，不在同一个slot槽位下的键值无法使用mset、mget等多键操作



可以通过{}来定义同一个组的概念，使key中{}内相同内容的键值对放到一个slot槽位去



CRC16源码浅谈


Redis集群有16384个哈希槽，每个key通过CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分hash槽。

常用命令
cluster-require-full-coverage
集群是否完整才对外提供服务



CLUSTER COUNTKEYSINSLOT slot
查看该槽号是否被占用

返回0，槽号尚未被占用
返回1，槽号已经被占用


CLUSTER KEYSLOT key
指定键应该存放到哪个槽位上


————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

原文链接：https://blog.csdn.net/qq_50082325/article/details/144493014