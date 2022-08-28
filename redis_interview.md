1. Redis 支持哪几种数据类型
    - hash的扩容收缩机制
        - 扩容为base * 2之后第一个 2^n的数
        - 渐进式rehash
            - rehashidx 置位0开始rehash, -1结束rehsh
            - 分而治之，ht[0]转移数据到ht[1]分散到增删改查操作上
            - 删，改，查先在ht[0]查找, 没有则查找ht[1]
            - 新增保存在ht[1]
    - rehash条件
        - loadfactor = ht[0].used / ht[0].size
        - loadfactor >= 1 并且没有执行 BGSAGE 或者 BGREWRITEAOF命令, 扩容
        - loadfactor >= 5 正在执行BGSAGE 或者 BGREWRITEAOF命令, 扩容
        - loadfactor < 0.1 收缩
2. Redis 达到配置上限后有哪几种数据淘汰策略
    - noeviction 不淘汰数据，大部分写操作会报错
    - volatitle-random 随机删除设置了ttl的键
    - volatitle-ttl 删除最接近ttl的键
    - volatitle-lru 通过lru算法删除设置了ttl的键
    - volatitle-lfu 通过lfu算法删除设置了ttl的键
    - allkeys-random 随机删除任意键
    - allkeys-lru 通过lru算法，删除任意键
    - allkeys-lfu 通过lfu算法，删除任意键
3. Redis删除ttl的键使用的策略
    - 可考虑的策略: 定时，惰性，定期
    - Redis采用的方案
        - 惰性：访问时查看是否过期，过期则删除
        - 定期：间隔一段时间，对ttl的键扫描删除
4. 一个字符串类型能存储的最大容量
5. redis适合场景？
    - 会话缓存（缓存购物车信息？)
    - 全页缓存(数据存入redis, 已最快速度加载浏览过的页面)
    - 队列
    - 排行榜/计数器
    - 发布/订阅
5. redis的集群方案
    - 主从 
6. 集群数据分片的支持
    1. 每个集群可以有 16384=2^14 个节点，每个集群可以有16384个槽(slot), 每个key属于一个特定的槽，每个key通过CRC16校验后对16384取模来决定放置哪个槽, 每个节点最多管理16384个槽
    2. 每个节点都会保存哪个槽归哪个节点负责
    3. 只能使用0号数据库
7. 高可用方案
    - 哨兵
8. 管道的作用
    - 使用管道技术可以显著提升Redis处理命令的速度，其原理就是将多条命令打包，只需要一次网络开销，在服务器端和客户端各一次read()和write()系统调用，以此来节约时间
9. redis的事务
    - 原子性
        - 入队时的命令报错，直接事务结束
        - exec执行时的报错，跳过
    - 隔离性
        - 串行执行
    - 一致性
    - 持久性
        - 无持久化策略，无持久性
        - RDB持久化, 无持久性
            - 满足条件才BGSAVE
            - BGSAVE是fork, cow的
        - AOF根据appendfsync配置不同则不同
            - always，有持久性(实际不怎么会用), 最多丢失一个事件循环的数据
                - no-appendfsync-on-rewrite打开时，无持久性, 默认关闭
            - everysec, 无持久性(该秒内宕机)
            - no, 无持久性(依赖操作系统刷盘)
        - 事务最后添加一条save之后在exec
            - 具备持久性，但效率太低，实际不采用
    - 推荐使用LUA脚本操作事务
        - 减少网络开销
        - 原子性
        - 替代redis的事务功能
10. redis事务相关的命令
    - multi 开启事务, 后续的命令入队
    - exec 依次执行队列命令
    - discard
    - watch
        - 在某些key满足一定条件下开启事务，需要通过watch监视key不改变key才能满足事务启动的条件
        - multi里无法写if条件判断(multi下的命令只是入队，exec才会依次执行队列命令)
11. RDB-AOF
    - RDB
        - SAVE阻塞主进程，BGSAVE先fork进程，通过cow技术让主进程继续工作(cow:主进程写时才复制一个内存页在上边写入，子进程在旧页上读取)，然后生成RDB文件
        - SAVE和BGSAVE不能同时执行
        - BGSAVE的默认设置(默认100ms检查一次是否能满足配置)
            - save 900 1
            - save 300 10
            - save 60 10000
        - BGSAVE执行时BGREWRITEAOF会排队; BGREWRITEAOF执行时，BGSAVE会拒绝；redis这样设计, 只是从性能的考虑(两个大量写磁盘的进程同时执行)
        - RDB文件在服务器启动时载入(没有特定命令载入RDB文件), 服务器启东优先检查AOF是否开启，开启则加载AOF文件，否则加载RDB文件, 载入时服务阻塞，直到载入结束
    - AOF(append only file)
        - 分为三步
            - 命令追加append, 写入缓冲区aof_buf
            - 文件写入
                always: 写入AOF文件，并执行同步
                everysec: 写入AOF文件，间隔达到1s，则执行同步
                no: 写入AOF文件，不执行同步，由操作系统控制同步(fsync, fdatasync)
            - 文件同步sync
    - AOF重写
        - 用压缩的AOF文件替换原AOF文件
        - 原理: 读取库中现状，生产对应指令，写入AOF文件(而不是解析原AOF文件)
        - 开辟子进程
            - 新的写入指令，追加到aof_buf, aof_rewrite_buf
            - 子进程结束后，给父进程发送信号，父进程将aof_rewrite_buf的内容写入新AOF文件, 父进程对新的AOF文件进行改名，原子地覆盖现有AOF文件
            - 只有信号处理阶段会阻塞主进程
12. 缓存穿透
    - 大量查询不存在的key, 会直接打在数据库上
    - 方案1, 结果为空的key也进行缓存，设置较小的ttl，查询后延期ttl
    - 方案2，布隆过滤器保存有效的key
13. 缓存击穿(缓存失效)
    - key过期时，大量请求该key，请求落在数据库上
    - 方案一：
        1. 查缓存失效时，加分布式锁更新数据
            - 加锁成功，读数据库更新缓存
            - 加锁失败, retry 从缓存get
        2. 不同的key, 设置不同过期时间, 缓存实效的时间分布均匀
    - 方案二:
        1. 查缓存失效时，加分布式锁更新数据
            - 单机DCL(单例模式的 double check lock)
            - 缺点：会阻塞其他的key
        2. 不同的key, 设置不同过期时间, 缓存实效的时间分布均匀
14. 缓存雪崩
    - 缓存崩溃，级联到数据库崩溃
    - 方案: 
        1. 缓存高可用, 哨兵、集群架构
        2. 通过隔离组件对后端限流，熔断，降级
            - 访问非核心数据，暂停从缓存查询，直接返回错误、空，预设的默认降级信息
            - 访问核心数据(库存), 仍然可从缓存查询，无缓存时查询数据库
        3. 缓存实效后，加锁或队列控制读取数据库写缓存的线程数
        4. 二级缓存，A1Redis缓存，本地A2拷贝缓存，A2通过独立的热点数据维护系统进行更新, 先访问A2，A2没有则访问A1
15. 分布式锁
    ```python
        def life_still(thread, time_sleep, callable):
            """锁续命"""
            while True:
                time.sleep(time_sleep / 3)  # todo: 优化为阻塞锁ttl的时长, 通过semaphore时间无CPU空转的阻塞
                if  thread.stop():  # 加锁的流程结束了
                    return
                ret = callable()
                if not ret:  # key不存在了
                    return

        
        product_id = 1
        lock_key = f"lock:product_id:{product_id}"

        # 不是原子的, 没设置val，可能会发生业务逻辑大于ttl时删除其他线程的锁的问题
        # result: int = redis.setnx(lock_key, "")

        # 原子的, val为线程特定
        client_id = str(uuid.uuid4())
        ttl = 30
        result: int = redis.setnx(lock_key, client_id, ttl, "seconds")  # 单纯延迟时间不能根本解决
        if not result:
            return

        # 不是原子的 
        # redis.expire(lock_key, 10, 'seconds')
        callable = lambda :redis.expire(lock_key, ttl)
        th = threading.Thread(target=life_still, args(Thread.current_thread, ttl, callable))
        try:
            total = redis.get('total')
            if total > 0:
                total -= 1
                redis.set('total', total)
            else:
                print("库存不足")
                return
        finally:
            if client_id = redis.get(lock_key):  # 查询与删除不是原子的，会导致执行del时间可能大于ttl, 删除其他线程的锁, 可以采用lua脚本解决
                # 9.9999s
                redis.del(lock_key)
    ```
    - 哨兵架构的分布式锁 REDLOCK(不推荐使用)
        - 半数以上的节点加锁成功，客户端才认为加锁成功
        - 没有百分百解决主从切换导致锁丢失的问题
            - 有从节点(主节点与从节点切换导致锁丢失)
            - 没有从节点, 需要引入更多主节点，防止半数的节点挂掉，节点越多需要加锁的节点就越多，性能降低, 不如使用zookeeper
            - 没有从节点, AOF的everysec内锁丢失，重启后没有此key
    - 优化
        - 减少锁粒度，将数据库一行数据，拆分为redis多个key(参考: ConcurrentHashmap java实现原理)
16. 异步队列
    1. list做队列，rpush生产消息，lpop消费消息，lpop无消息时，需要sleep
    2. 缺点？
    3. 1:N消费，使用 pub/sub 主题订阅
17. 延时队列
    1. sortedset，时间戳作score, 消息内容为key，zadd生产消息, zrangebyscore获取n秒之前的数据轮询处理
    2. 使用kafka实现延迟队列
18. 扫描可以，1亿key，扫描10w个prefix相同的key
    - keys 列出指定模式的key
    - redis单线程，keys阻塞正常业务
    - 使用scan(基于游标)扫描指定模式的key, 但是会重复或缺失, 客户端做去重
    - scan花费时间比keys长(没有确定好原因)
        - scan 新老dict都要扫描
        - keys 全量扫
19. 常见的性能问题
    - master 最好不要做持久化工作，如 RDB 内存快照和 AOF 日志文件
    - 如果数据比较重要，某个 slave 开启 AOF 备份，策略设置成每秒同步一次
    - 为了主从复制的速度和连接的稳定性，master 和 Slave 最好在一个局域网内
    - 尽量避免在压力大得主库上增加从库
    - 主从复制不要采用网状结构，尽量是线性结构，Master<--Slave1<----Slave2 ....
20. 缓存架构
    - 新增缓存(缓存重建)时使用分布式锁, 空缓存或None, 直接返回
        - 可以权衡的优化：trylock(waittime), 避免大量的请求都要重复执行加锁释放的逻辑，但是waittime需要依赖锁住的业务代码, 实际可能不好把控，常设置1s
    - 新增缓存必须有ttl, ttl增加随机数(24h)
    - 新增缓存且数据库为空也进行缓存(空缓存), 并且设置短的ttl, 查询进行短延期
    - 查询缓存时做ttl延期
21. 缓存与数据库双写/读写不一致
    - 原因: 更新数据库与更新缓存不是原子的
    - 方案: 加分布式锁，更新数据库+更新缓存，释放锁
    - 优化: 分布式读写锁，只读的操作加读锁(可重入锁)，写入数据加写锁, 读写锁是同一个key(hash类型), 读写冲突的效果是等待对方
22. 主从数据库不一致如何解决