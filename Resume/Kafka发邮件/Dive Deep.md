# 整体
![image](./img/整体.png)

## 思考
### Why Kafka?
* 问题场景
    * SMTP服务器限制每分钟只能发送30封邮件
    * 但业务系统可能在瞬间产生大量邮件请求(比如:用户注册验证码、订单通知等)
    * 直接发送可能超过SMTP限制导致失败
* Kafka的解决方案
    * 业务系统将邮件请求发送到Kafka (无速率限制)
    * 消费者以固定速率(比如每分钟30封)从Kafka消费并通过SMTP发送
    * 超出SMTP处理能力的请求会在Kafka中缓存，而不会丢失
* 效果
    * 业务系统可以快速处理大量邮件请求(高throughput)
    * SMTP发送速率保持在限制之内
    * 系统整体吞吐量得到提升，同时保证可靠性
### 具体实现流程
* Producer端
    * 比如秒级产生1000封邮件请求
    * 快速写入Kafka（毫秒级）
    * 业务系统无需等待邮件发送完成
    * 实现了高吞吐（high throughput）
* Kafka中间层
    * 可以缓存大量邮件消息
    * 消息持久化，保证可靠性
    * 支持消息堆积，不会丢失
* Consumer端
    * 使用RateLimiter控制速率为每分钟30封
    * 符合SMTP服务器限制
    * 平稳发送，不会导致服务崩溃

### 职责和impact
* 负责邮件发送系统的架构设计和开发。通过Kafka消息队列实现了完整的邮件处理流程：

设计了三层队列架构：主发送队列(send-email)、重试队列(email-retry)和死信队列(email-dead-letter-queue)
在Consumer端实现了RateLimiter限流机制，有效控制SMTP服务器的发送频率
实现了完整的异常处理和重试机制，大大减少了throttle exception的发生
引入dead-letter-queue处理最终失败的邮件，提升了系统的可观测性

这个架构显著提升了系统的吞吐量和可靠性，同时保证了邮件服务的稳定性。"

![image](./img/整体架构图.png)

### 分布式
* 三个server，三个smtp，一个kafka，在producer端进行partition和随机发送到不同的partition
* Key不指定，随机分配
* 每个Partition能保证fifo，整体不保证
