1. kafka发送到指定分区
    - 通过key
    - 通过执行patation
2. kafka发送消息持久化机制
    - producer.acks_config = 1
    - acks==0, producer不需要等待broker的ack，就可以继续发消息，容易丢消息
    - acks==1, producer需要等待leader将数据写入log，但不需要follower成功写入，就可以继续发消息。如果follower没有成功备份到log，leader又挂了，就会丢消息
    - acks==-1/all, producer需要等待min.insync.replicas(默认为1, 推荐大于等于2)个follower成功备份到log。金融级别使用
3. kafka重要配置
    - producer.retries_config = 3
        - 发送失败会重试，默认重试间隔100ms，重试能保证发送消息的可靠性，但是可能会造成消息重复发送, 消费者要做好消息幂等的处理
    - producer.retry_backoff_ms_config = 300
        - 重试间隔 300ms
    - producer.buffer_memory_config = 33554432(32MB)
        - 先发送到buffer, 可以提高消息发送性能
    - producer.batch_size_config = 16384(KB)
        - kafka本地线程从缓冲区读取16kb到batch中，发送一次到broker
        - 默认是0，表示消息立即发送，但会影响性能
    - producer.linger_ms_config = 10(ms)
        - 10ms内batch没填满16kb也需要立即发送到broker
    - consumer.enable_auto_commit_config = true
        - 控制commit
        - 业务中设为false
    - consumer.auto_commit_interval_ms_config = 1000
        - 自动提交为true时, commit的等待时间
    - consumer.poll(duration)
        - 长轮询: duration时间内没拉到消息，会再次拉取，直到拉取时间大于duration, 返回空
        - 拉下来一批数据，处理完发送一次commit
    - consumer.max_poll_records_config = 500
        - 一次poll最多拉500条数据
    - consumer.max_poll_interval_ms_config = 30 * 1000
        - 如果两次poll操作间隔超过此时间，broker将此consumer踢出该消费者组, 将分区分配给其他消费者组
    - consumer.session_timeout_ms_config = 10 * 1000
        - consumer与broker的心跳时间间隔
        - 超过该时间, broker将此consumer踢出该消费者组, 将分区分配给其他消费者组
    - consumer.heartbeat_interval_ms_config = 1000
        - 多久发送一次心跳
    - consumer.auto_offset_reset_config = 'earliest'
        - earliest, 区别于seek_from_beginning, 从上次offset开始消费
        - latest, 从启东时的offset开始消费
        - none, 只要有一个分区没有offset报错，否则从offset开始消费
    - consumer.commit_sync()
        - 同步提交
    - consumer.commit_async()
        - 异步提交
4. 消费者
    - 指定消费开始的offset, consumer.seek
    - 指定消费开始的时间, consumer.offsets_for_tiems() 之后 consumer.seek
5. 日志分段存储
    - 默认上限是1GB切换到下一个文件
    - 000.index
    - 000.log
    - 000.timeindex
    - 050.index
    - 050.log
    - 050.timeindex
5. partition的leader选举
6. 消费者rebalance的条件
    - 消费者组里的consumer增加货减少
    - 动态给topic增加分区
    - 消费者订阅了更多topic
    - rebalance过程中，消费者无法从kafka消费消息，对kafka的TPS有影响
7. rebalance策略
    - range(default): n = 分区数/消费者数 m = 分区数%消费者数，前m个消费者分配n+1个分区，后边的消费者数-m分配n个分区
    - round-robin
    - sticky(与round-robin类似)
        - 分区的分配尽可能均匀
        - 分区的分配尽可能与上次分配的保持相同
        - 两个冲突时，第一目标优先于第二目标
8. HW(high watermark) LEO(log-end-offset)
    - ISR中最小的LEO作为HW，保证消费数据的一致性
9. 消息丢失
    - producer
    - consumer
10. 消息重复消费
    - producer配置了重试机制
    - consumer自动提交
        - 消费幂等处理
11. 消息乱序
    - 配置了重试机制 1, 2, 3发送，1失败重试，变成了2, 3, 1
    - 方案: 消费端自己实现逻辑
12. 消息积压
    - 消费者太少，由于消费者是绑定分区的，新增消费者是不能处理消息积压(rebalance会阻塞消费，不能采纳)
        - 将其中一个消费者逻辑改为转发逻辑到新的topic，增加大量分区，启东大量消费者
    - 逻辑bug导致消息消费不了
        - 创建死信队列，将不能消费的消息转发到此topic下，后面慢慢分析 
13. 延时队列
    - 需要自己实现或者使用rocketmq
14. 生产者幂等
    - 鸡肋，一般需要在消费端做幂等处理
    -  producer.enable_idempotence = true 默认false
15. kafka事务
    - 流式计算, 一条消息发送到不同的topic, 都成功/都失败
16. kafka高性能原因
    - 日志顺序读写
    - 数据传输的零拷贝
        - 通过操作系统的sendfile实现零拷贝(MMAP: 用户态引用内核态)
        - 减少了2次数据从内核空间复制到用户空间
        - 减少了内核态与用户态上下文切换
    - 读写数据的批量batch处理及压缩传输