# Situation
## 项目背景和目标
* 痛点: 公司不同的业务开发组在使用thread pool都有一个共同的痛点 -- 
1. 线程池的配置过度依赖开发人员的个人经验
2. 不同的业务类型对线程资源的需求差异很大。比如在儿童保险业务中:
* 邮件发送是I/O密集型, 核心线程数较多：通常是CPU核心数的2倍以上 + 最大线程数可以更大：比如CPU核心数的4倍. 原因：IO线程大多在等待，适合多开线程提高吞吐量
* 保险费率计算是CPU密集型, 核心线程数较少,通常等于CPU核心数; 最大线程数接近核心数：一般是CPU核心数+1。原因：避免过多线程竞争CPU资源，减少上下文切换
## 为什么需要这个项目（业务价值）
* 之前的业务线中, 每次修改线程池参数后需要重新发布==>系统不够灵活
* 解决方案: 降低线程池修改的成本，将线程池的参数从代码中迁移到分布式配置中心,实现了线程池配置可动态配置和即时生效

# Task
## 你在团队中的角色/具体负责哪些部分
* 我独立开发设计了 Dynamic Thread Pool这个组件，实现corePoolSize、maximumPoolSize， workQueue的动态调整，并上传到Azure DevOps Artifacts上实现了跨团队复用
* 使用Redis作为注册中心，实现不同线程池信息的统一管理。通过发布/订阅功能，实现线程池的更改。
* 开发了前端页面和admin管理中心，并与运维团队合作，实现了Prometheus监控和告警 + 创建了Granafa面板。监控了active_count，maximumPoolSiz，queue size，completed task 等指标。设置了活跃度报警: active/poolSize > 0.8等告警rules
# Actions
## 有哪些模块？
### Config模块
- 所以Config模块确实是整个组件的入口点，负责：
* Redis连接的建立(RedissonClient)
* 线程池配置的初始化
* 组件各个部分的协调和启动
* 配置的自动装配
### Domain模块
* 提供了线程池query和update
### Registry模块
* 负责线程池配置信息在Redis中的存储和管理
### Trigger模块
* 定时任务，负责采集和上报线程池运行时数
* 配置变更监听器，监听Redis的配置更新事件并触发线程池参数动态调整
## 技术设计和决策
### 为什么不自动扩容？
* 保险业务通常比较稳定，日常claims处理量可预测
* 突发情况通常有预警，在开学季会提前扩容（7-9）
* 降低系统复杂度
* 减少运维负担
## 实现过程中遇到的挑战和解决方案
### 需要确定到底监控的指标是什么，需要理解线程池的核心参数
* active count: 活跃线程数
* max size: 最大线程数
* queue size: 队列大小
* completed task: 已完成任务数
* reject count: 拒绝任务数
* execute time: 执行时间
### 如果设置的core thread数量小于当前 active的thread count会怎样？
* 不会中断正在执行的线程，而是会等待，处理完一个任务后，进行检查，如果超过就回收
* 不建议调小，除非长期的(e.g. 3 months)都闲置，减少监控的压力
## 跨团队协作的细节
### 运维团队
* Prometheus和Grafana的接入配置:公司一套统一的监控平台地址
* Admin部署环境信息: 
* Admin端需要的Redis registry连接信息: xxx.redis.cache.azure.net:6380
### 开发团队
* 告警系统的比例设置
    * 活跃度报警: active/poolSize > 0.8
    * 拒绝率报警: > 5%
    * 队列使用率: queueSize / queueCapacity > 0.8
* 写了详细的开发文档，上传到公司的Confluence中，提供了详细的组件使用手册

# Result
* 极大提高了复用率，在公司的travel/truck/children insurance业务中都有使用
* 提高了团队开发效率和系统处理突发事件的能力
* 收获了很多组件开发，problem-sloving, 团队合作的能力

