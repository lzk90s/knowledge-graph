<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1. 分布式基础理论](#1-分布式基础理论)
  - [1.1. 分布式 CAP 定理](#11-分布式-cap-定理)
  - [1.2. 一致性的三种类型](#12-一致性的三种类型)
  - [1.3. BASE 理论](#13-base-理论)
  - [1.4. 分布式一致性 hash 算法](#14-分布式一致性-hash-算法)
- [2. 分布式 ID](#2-分布式-id)
  - [2.1. 分布式唯一 id 生成方法](#21-分布式唯一-id-生成方法)
  - [2.2. snowflake 雪花算法原理](#22-snowflake-雪花算法原理)
  - [2.3. 雪花算法存在的问题](#23-雪花算法存在的问题)
  - [2.4. 美团 leaf](#24-美团-leaf)
  - [2.5. 百度 uid-generator](#25-百度-uid-generator)
- [3. 分布式事务](#3-分布式事务)
  - [3.1. 两阶段提交 2PC、三阶段提交 3PC](#31-两阶段提交-2pc-三阶段提交-3pc)
  - [3.2. 事务补偿 TCC：一个接口拆成 Try、Confirm、Cancel3 个](#32-事务补偿-tcc一个接口拆成-try-confirm-cancel3-个)
  - [3.3. 本地消息表（最终一致性）](#33-本地消息表最终一致性)
  - [3.4. MQ 事务消息](#34-mq-事务消息)
  - [3.5. seata 分布式事务框架](#35-seata-分布式事务框架)
- [4. 分布式锁](#4-分布式锁)
  - [4.1. 基于数据库](#41-基于数据库)
  - [4.2. 基于 redis](#42-基于-redis)
  - [4.3. redis 分布式锁的注意点（坑）](#43-redis-分布式锁的注意点坑)
  - [4.4. 基于 zookeeper](#44-基于-zookeeper)
- [5. 分布式系统选举算法](#5-分布式系统选举算法)
  - [5.1. Bully 算法](#51-bully-算法)
  - [5.2. Paxos 算法是什么？](#52-paxos-算法是什么)
  - [5.3. Raft 算法是什么？](#53-raft-算法是什么)
- [6. 分布式系统常见问题](#6-分布式系统常见问题)
  - [6.1. 幂等操作是什么意思？](#61-幂等操作是什么意思)
  - [6.2. 如何实现接口的幂等？避免重复提交](#62-如何实现接口的幂等避免重复提交)

<!-- /code_chunk_output -->

## 1. 分布式基础理论

### 1.1. 分布式 CAP 定理

1. 针对的是多个节点之间的读写一致提出的
2. （Consistency）一致性 分布式系统完成某个写操作后任何读操作，都应该获取到该写操作写的的最新值，相当于要求分布式系统中的各节点时刻保持数据的一致性
3. （Availability）可用性 一直可以正常的做读写操作，对用户而言不会出现操作失败或者访问超时
4. （Partition tolerance）分区容错性 指某个节点或者网络出现故障时导致各个节点数据不一致还能返回数据

### 1.2. 一致性的三种类型

1. 强一致性 数据更新成后任意时刻所有副本的数据都一致，一般采用同步方式实现
2. 弱一致 数据更新成功后不承诺多久读到新值，也不承诺多久能读到
3. 最终一致 更新成功后，不承诺立即返回新值，但保证最终会返回新值

### 1.3. BASE 理论

1. BA （Basically Available）指的是基本业务可用性，支持分区失败
2. S （ Soft State）表示柔性状态，也就是允许短时间内不同步
3. E （ Eventual Consistency）表示最终一致性，但是实时不一定一致

### 1.4. 分布式一致性 hash 算法

## 2. 分布式 ID

### 2.1. 分布式唯一 id 生成方法

| 方法               | 描述                                                                                                                             | 优点                                                                                                       | 缺点                                                                                   |
| :----------------- | :------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------- |
| uuid               | 算法的核心思想是结合机器的网卡、当地时间、一个随记数来生成 UUID                                                                  | 本地生成，生成简单，性能好，没有高可用风险                                                                 | 长度过长，存储冗余，且无序不可读，查询效率低                                           |
| 数据库自增 id      | 使用数据库的 id 自增策略，如 MySQL 的 auto_increment。并且可以使用两台数据库分别设置不同步长，生成不重复 ID 的策略来实现高可用。 | 数据库生成的 ID 绝对有序，高可用实现方式简单                                                               | 需要独立部署数据库实例，成本高，有性能瓶颈                                             |
| 批量申请自增 ID    | 一次按需批量生成多个 ID，每次生成都需要访问数据库，将数据库修改为最大的 ID 值，并在内存中记录当前值及最大值                      | 避免了每次生成 ID 都要访问数据库并带来压力，提高性能                                                       | 属于本地生成策略，存在单点故障，服务重启造成 ID 不连续                                 |
| redis incr/incrby  | Redis 的所有命令操作都是单线程的，本身提供像 incr 和 increby 这样的自增原子命令，所以能保证生成的 ID 肯定是唯一有序的            | 不依赖于数据库，灵活方便，且性能优于数据库；数字 ID 天然排序，对分页或者需要排序的结果很有帮助             | 如果系统中没有 Redis，还需要引入新的组件，增加系统复杂度；需要编码和配置的工作量比较大 |
| snowflake 雪花算法 | 使用一个 64 bit 的 long 型的数字作为全局唯一 id。在分布式系统中的应用十分广泛，且 ID 引入了时间戳，保持自增性且不重复            | 整体上按照时间自增排序，并且整个分布式系统内不会产生 ID 碰撞(由数据中心 ID 和机器 ID 作区分)，并且效率较高 | 依赖机器时钟，如果时钟回调，会导致冲突                                                 |

### 2.2. snowflake 雪花算法原理

![snowflake](snowflake.png)

- 1 bit，是无意义的：因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0。
- 41 bit：表示的是时间戳，单位是毫秒。41 bit 可以表示的数字多达 2^41 - 1，也就是可以标识 2 ^ 41 - 1 个毫秒值，换算成年就是表示 69 年的时间。
- 10 bit：记录工作机器 id，代表的是这个服务最多可以部署在 2^10 台机器上，也就是 1024 台机器。但是 10 bit 里 5 个 bit 代表机房 id，5 个 bit 代表机器 id。意思就是最多代表 2 ^ 5 个机房（32 个机房），每个机房里可以代表 2 ^ 5 个机器（32 台机器），这里可以随意拆分，比如拿出 4 位标识业务号，其他 6 位作为机器号。可以随意组合。
- 12 bit：这个是用来记录同一个毫秒内产生的不同 id。12 bit 可以代表的最大正整数是 2 ^ 12 - 1 = 4096，也就是说可以用这个 12 bit 代表的数字来区分同一个毫秒内的 4096 个不同的 id。也就是同一毫秒内同一台机器所生成的最大 ID 数量为 4096

假设某个服务要生成一个全局唯一 id，那么就可以发送一个请求给部署了 SnowFlake 算法的系统，由这个 SnowFlake 算法系统来生成唯一 id。这个 SnowFlake 算法系统首先肯定是知道自己所在的机器号，接着 SnowFlake 算法系统接收到这个请求之后，首先就会用二进制位运算的方式生成一个 64 bit 的 long 型 id，64 个 bit 中的第一个 bit 是无意义的。接着用当前时间戳（单位到毫秒）占用 41 个 bit，然后接着 10 个 bit 设置机器 id。最后再判断一下，当前这台机房的这台机器上这一毫秒内，这是第几个请求，给这次生成 id 的请求累加一个序号，作为最后的 12 个 bit。

### 2.3. 雪花算法存在的问题

1. workid 需要手动分配
2. 时钟回拨导致 id 重复问题

### 2.4. 美团 leaf

### 2.5. 百度 uid-generator

## 3. 分布式事务

### 3.1. 两阶段提交 2PC、三阶段提交 3PC

两阶段提交（2PC） 引入协调者（Coordinator）来协调参与者的行为，并最终决定这些参与者是否要真正执行事务。

- 准备阶段：协调者询问参与者事务是否执行成功，参与者发回事务执行结果。
- 提交阶段：如果事务在每个参与者上都执行成功，事务协调器者发送通知让参与者提交事务，否则协调者发送通知让参与者回滚事务。

在准备阶段，参与者执行了事务但是未提交，只有在提交阶段收到协调者发来的通知后才进行提交或者回滚。

缺点：

- 单点问题：如果事务管理器出现故障，资源管理器将一直处于锁定状态。
- 性能问题：所有资源管理器在事务提交阶段处于同步阻塞状态，占用系统资源，一直到提交完成，才释放资源，容易导致性能瓶颈。
- 数据一致性问题：如果有的资源管理器收到提交 commit 的消息，有的没收到，那么会导致数据不一致问题。

- PreCommit
- Commit

三阶段提交相比二阶段提交有两个改动点：

- 引入超时机制。同时在协调者和参与者中都引入超时机制。
- 在第一阶段和第二阶段中插入一个准备阶段。保证了在最后提交阶段之前各参与节点的状态是一致的。

三阶段提交其实没根本性解决问题，它仅仅是引入了 perpared commit 阶段，无法解决单点故障或网络脑裂问题，仅仅是多做了一次校验而已，治标不治本，无法彻底解决分布式一致性。

### 3.2. 事务补偿 TCC：一个接口拆成 Try、Confirm、Cancel3 个

补偿事务（TCC） 针对每个操作，都要注册一个与其对应的确认和补偿（撤销）操作，分为三个阶段：

- Try 阶段主要是对业务系统做检测及资源预留；
- Confirm 阶段主要是对业务系统做确认提交，Try 阶段执行成功并开始执行 Confirm 阶段时，默认 Confirm 阶段时不会出错的，即只要 Try 成功 Confirm 一定成功；
- Cancel 阶段主要是在业务执行错误，需要回滚的状态下执行的业务取消，预留资源释放。

优点：跟 2PC 比起来，实现以及流程相对简单些，但是数据执行比 2PC 也要差一些，缺点：2,3 两步都有可能失败，要写很多代码保证

### 3.3. 本地消息表（最终一致性）

此方案的核心是将需要分布式处理的任务通过消息日志的方式来异步执行。消息日志可以存储到本地文本、数据库或消息队列，再通过业务规则自动或人工发起重试。

优点：避免了分布式事务，实现了最终一致性
缺点：消息表会耦合到业务系统中，如果没有封装好的解决方案会增加工作量。

### 3.4. MQ 事务消息

有些 MQ 是支持事务消息的，如 RocketMQ，他们支持事务消息的方式类似于二阶段提交。

1. 第一阶段 Prepared 消息，会拿到消息的地址；
2. 第二阶段执行本地事务；
3. 第三阶段通过第一阶段拿到的地址去访问消息，并修改状态。也就是说在业务方法内要消息队列提交两次请求，一次发送消息和一次确认消息。如果确认消息发送失败了 RocketMQ 会定期扫描消息集群中的事务消息，这时候发现了 Prepared 消息，会向消息发送者确认，所以生产方要实现一个 check 接口，RocketMQ 会根据发送端设置的策略来决定是回滚还是继续发送确认消息。这样就保证了消息发送与本地事务同时成功或者同时失败。

优点：实现了最终一致性，不需要依赖本地数据库事务
缺点：大多数 MQ 不支持

### 3.5. seata 分布式事务框架

见 [seata](../middleware/seata.md)

## 4. 分布式锁

### 4.1. 基于数据库

实现方式：利用的是乐观锁和悲观锁

- 乐观锁：在表中添加版本号的字段，每次更新前都先查询出带版本号的数据，然后再更新的时候 where 条件语句后带版本号条件，更新成功表示锁已占用，更新不成功表示锁没被占用。
- 悲观锁：利用 select...for update（X 锁）/select...lock in share mode（S 锁），一般来说用 X 锁的较多，因为后续多会做写功能的实现。

缺点：

### 4.2. 基于 redis

redis 实现分布式锁主要靠四个命令：

1. setnx（set if not exits 维护着是乐观锁）：当不存在 key 的时候，才为 key 设置值为 value。setnx 与 set 的区别：set 是存在 key，则去覆盖 value；setnx 是不存在 key，则重新给 key 和 value 赋值。
2. getset：根据 key 得到旧的值，并 set 新的值。
3. expire：设置过期时间。
4. del：删除

实现方式

1. 获取锁的时候，使用 setnx 加锁，并使用 expire 命令为锁添加一个超时时间，超过该时间则自动释放锁，锁的 value 值为一个随机生成的 UUID，通过此在释放锁的时候进行判断。
2. 获取锁的时候还设置一个获取的超时时间，若超过这个时间则放弃获取锁。
3. 释放锁的时候，通过 UUID 判断是不是该锁，若是该锁，则执行 delete 进行锁释放。

### 4.3. redis 分布式锁的注意点（坑）

一、多线程下，锁被其他线程释放了

A、B 两个线程来尝试给 key myLock 加锁，如果此时业务逻辑比较耗时，执行时间已经超过 redis 锁过期时间，这时 A 线程的锁自动释放（删除 key），B 线程检测到 myLock 这个 key 不存在，执行 SETNX 命令也拿到了锁。但是，此时 A 线程执行完业务逻辑之后，还是会去释放锁（删除 key），这就导致 B 线程的锁被 A 线程给释放了。为避免上边的情况，一般我们在每个线程加锁时要带上自己独有的 value 值来标识，只释放指定 value 的 key，否则就会出现释放锁混乱的场景。

二、锁过期了，业务还没执行完

给 redis 锁的过期时间自动续期，可以使用 redisson 客户端来操作 redis,redisson 封装了分布式锁，redisson 在加锁成功后，会注册一个定时任务监听这个锁，每隔 10 秒就去查看这个锁，如果还持有锁，就对过期时间进行续期。默认过期时间 30 秒。

例如：加锁的时间是 30 秒，过 10 秒检查一次，一旦加锁的业务没有执行完，就会进行一次续期，把锁的过期时间再次重置成 30 秒

三、在数据库事务中获取锁，获取锁的时间很长，数据库事务超时

解决这种问题，可以将数据库事务改为手动提交、回滚事务。

### 4.4. 基于 zookeeper

Zookeeper：利用 Zookeeper 的顺序临时节点，来实现分布式锁和等待队列。Zookeeper 设计的初衷，就是为了实现分布式锁服务的。

利用临时顺序节点实现共享锁:

算法：对于加锁操作，可以让所有客户端都去/lock 目录下创建临时顺序节点，如果创建的客户端发现自身创建节点序列号是/lock/目录下最小的节点，则获得锁。否则，监视比自己创建节点的序列号小的节点（比自己创建的节点小的最大节点），进入等待。
比如创建节点：/lock/0000000001、/lock/0000000002、/lock/0000000003。则节点/lock/0000000001 会先获得锁，因为 zk 上的节点是有序的，且都是最小的节点先获得锁。

为什么使用临时目录？是因为临时目录在进程挂了后会自动删除，释放锁。
为什么只监听比自己小的最后一个节点？是为了避免惊群现象。和 AQS 类似的队列

## 5. 分布式系统选举算法

### 5.1. Bully 算法

在所有活着的节点中，选取 ID 最大的节点作为主节点。

### 5.2. Paxos 算法是什么？

Paxos 算法解决的问题是在一个可能发生异常（进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复、不考虑篡改）的分布式系统中如果就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性

### 5.3. Raft 算法是什么？

Raft 算法是典型的多数派投票选举算法，其选举机制与我们日常生活中的民主投票机制类似，核心思想是“少数服从多数”。也就是说，Raft 算法中，获得投票最多的节点成为主。

采用 Raft 算法选举，集群节点的角色有 3 种：

- Leader，即主节点，同一时刻只有一个 Leader，负责协调和管理其他节点；
- Candidate，即候选者，每一个节点都可以成为 Candidate，节点在该角色下才可以被选为新的 Leader；
- Follower，Leader 的跟随者，不可以发起选举。

Raft 选举的流程，可以分为以下几步：

1. 初始化时，所有节点均为 Follower 状态。
2. 开始选主时，所有节点的状态由 Follower 转化为 Candidate，并向其他节点发送选举请求。
3. 其他节点根据接收到的选举请求的先后顺序，回复是否同意成为主。这里需要注意的是，在每一轮选举中，一个节点只能投出一张票。
4. 若发起选举请求的节点获得超过一半的投票，则成为主节点，其状态转化为 Leader，其他节点的状态则由 Candidate 降为 Follower。Leader 节点与 Follower 节点之间会定期发送心跳包，以检测主节点是否活着。
5. 当 Leader 节点的任期到了，即发现其他服务器开始下一轮选主周期时，Leader 节点的状态由 Leader 降级为 Follower，进入新一轮选主

每一轮选举，每个节点只能投一次票。这种选举就类似人大代表选举，正常情况下每个人大代表都有一定的任期，任期到后会触发重新选举，且投票者只能将自己手里唯一的票投给其中一个候选者。对应到 Raft 算法中，选主是周期进行的，包括选主和任值两个时间段，选主阶段对应投票阶段，任值阶段对应节点成为主之后的任期。但也有例外的时候，如果主节点故障，会立马发起选举，重新选出一个主节点。

## 6. 分布式系统常见问题

### 6.1. 幂等操作是什么意思？

任意多次执行所产生的影响均与一次执行的影响相同。

### 6.2. 如何实现接口的幂等？避免重复提交

1. 数据库主键校验唯一 id 方式，重复提交报错
2. token 方式，先向服务端申请一个 token，服务端把 token 存到 redis 中,业务调用时把 token 传到服务端。调用接口后，删除 redis 中的 token。校验请求，当 redis 中存在 token 时，表示是第一次调用，不存在时，表示是重复提交。
3. redis setnx 方式，用一个唯一键 setnx，第一次可以成功，后续的提交会失败，setnx 需要设置一个超时时间。
