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
- 所以Config模块确实是整个组件的入口点(**entry point**)，负责：
* Redis连接的建立(RedissonClient)
* 线程池配置的初始化 (Initializing thread pool configurations)
* 组件各个部分的协调和启动 (Coordinating and starting different parts of the component)
* 配置的自动装配 (Auto-configuration setup)
### Domain模块
* 提供了线程池query和update (query/update operations)
### Registry模块
* 负责线程池配置信息在Redis中的存储和管理 (storage and management)
### Trigger模块
* 定时任务，负责采集和上报线程池运行时数 (Scheduled tasks, **collecting and reporting** thread pool runtime metrics)
* 配置变更监听器，监听Redis的配置更新事件并触发线程池参数动态调整 (Configuration change listener, monitoring Redis configuration update events and triggering dynamic **adjustments** of thread pool parameters)
## 技术设计和决策
### 为什么不自动扩容？(automatic scaling?)
* 保险业务通常比较稳定，日常claims处理量可预测 (stable, with predictable daily claims processing volume)
* 突发情况通常有预警，在开学季会提前扩容（7-9）(**Sudden spikes** usually come with advance warning,school season)
* 降低系统复杂度 (Reduces system complexity)
* 减少运维负担 (Decreases **operational maintenance** burden)
## 实现过程中遇到的挑战和解决方案
### 需要确定到底监控的指标是什么，需要理解线程池的核心参数
* active count: 活跃线程数
* max size: 最大线程数
* queue size: 队列大小
* completed task: 已完成任务数
* reject count: 拒绝任务数
* execute time: 执行时间
### 如果设置的core thread数量小于当前 active的thread count会怎样？
* 不会中断正在执行的线程，而是会等待，处理完一个任务后，进行检查，如果超过就回收 (Does not interrupt currently executing threads, but rather waits and checks after each task completion, recycling if exceeding the limit)
* 不建议调小，除非长期的(e.g. 3 months)都闲置，减少监控的压力 (Downsizing is not recommended unless there has been long-term **idle capacity** (e.g., 3 months), to reduce monitoring pressure)
## 跨团队协作的细节
### 运维团队
* Prometheus和Grafana的接入配置:公司一套统一的监控平台地址 （Company's unified monitoring platform address）
* Admin部署环境信息: (Admin deployment environment information)
* Admin端需要的Redis registry连接信息: xxx.redis.cache.azure.net:6380 (Redis registry connection information required for Admin side: xxx.redis.cache.azure.net:6380)
### 开发团队
* 告警系统的比例设置
    * 活跃度报警: active/poolSize > 0.8 (Thread pool **utilization rate**)
    * 拒绝率报警: > 5%   
    * 队列使用率: queueSize / queueCapacity > 0.8 
* 写了详细的开发文档，上传到公司的Confluence中，提供了详细的组件使用手册

# Result
* 极大提高了复用率，在公司的travel/truck/children insurance业务中都有使用
* 提高了团队开发效率和系统处理突发事件的能力
* 收获了很多组件开发，problem-sloving, 团队合作的能力


# 动态线程池项目面试准备

## Situation

### 项目背景和目标

* 痛点：公司不同业务开发组在使用线程池时面临共同挑战
  * 线程池配置过度依赖开发人员个人经验
  * 不同业务类型对线程资源需求差异大
    * 邮件发送（I/O密集型）
      * 核心线程数：CPU核心数×2以上
      * 最大线程数：CPU核心数×4
      * 原因：IO线程多在等待，适合多开线程提高吞吐量
    * 保险费率计算（CPU密集型）
      * 核心线程数：等于CPU核心数
      * 最大线程数：CPU核心数+1
      * 原因：避免线程过多竞争CPU，减少上下文切换

### 业务价值

* 提升系统灵活性
  * 之前：修改线程池参数需重新发布
  * 现在：参数迁移至分布式配置中心，支持动态调整和即时生效
* 降低运维成本
  * 统一的监控平台
  * 可视化的配置管理
* 提高系统可靠性
  * 精确的监控指标
  * 及时的告警机制

## Task

### 个人角色和职责

* 技术负责人
  * 独立设计开发Dynamic Thread Pool组件
  * 实现核心功能和跨团队复用
  * 开发管理界面和监控系统
  * 编写技术文档和使用手册

### 核心功能开发

* 组件设计和实现
  * 支持动态调整：corePoolSize、maximumPoolSize、workQueue
  * 发布到Azure DevOps Artifacts实现复用
* 配置中心集成
  * 使用Redis作为注册中心
  * 实现配置的统一管理
  * 通过发布/订阅实现实时更新
* 监控系统搭建
  * 对接Prometheus
  * 创建Grafana面板
  * 设置关键指标告警

## Action

### 模块设计

1. Config模块（核心入口）
   * Redis连接管理
   * 线程池配置初始化
   * 组件协调和启动
   * 自动装配支持

2. Domain模块（业务核心）
   * 线程池查询接口
   * 线程池更新操作
   * 参数验证和转换

3. Registry模块（配置中心）
   * Redis数据存储
   * 配置信息管理

4. Trigger模块（监控触发）
   * 数据采集定时任务
   * 配置变更监听
   * 参数动态调整

### 技术决策

* 为什么选择手动而非自动扩容？
  * 业务特点：保险业务较稳定，处理量可预测
  * 运维考虑：降低系统复杂度，减少维护负担
  * 实际需求：特殊时期（如开学季）可提前人工扩容

### 挑战与解决方案

1. 监控指标设计
   * 活跃线程数（active count）
   * 最大线程数（max size）
   * 队列大小（queue size）
   * 已完成任务数（completed task）
   * 拒绝任务数（reject count）
   * 执行时间（execute time）

2. 动态调整策略
   * 核心线程收缩：等待任务完成后自然回收
   * 建议：除非长期闲置（3个月以上），不建议调小配置

### 跨团队协作

1. 运维团队协作
   * 监控平台接入
   * 部署环境配置
   * Redis连接信息维护

2. 开发团队支持
   * 告警阈值设置
     * 活跃度：active/poolSize > 0.8
     * 拒绝率：> 5%
     * 队列使用率：queueSize/queueCapacity > 0.8
   * 文档支持：Confluence上的使用手册

## Result

### 项目成果

* 业务成果
  * 提高系统可用性和稳定性
  * 降低运维成本和人工干预
  * 提升配置灵活性和响应速度

* 技术成果
  * 在多个业务线成功应用（travel/truck/children insurance）
  * 建立了统一的线程池管理标准
  * 提供了可复用的技术组件

### 个人收获

* 技术能力
  * 组件设计和开发经验
  * 分布式系统实践
  * 监控系统搭建经验

* 软技能
  * 跨团队协作能力
  * 技术文档编写
  * 问题分析和解决能力

### 可改进点

* 配置验证：增加更严格的参数校验
* 监控覆盖：扩展更多业务相关指标
* 故障恢复：完善配置回滚机制