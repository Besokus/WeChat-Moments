# 生产就绪检查清单

## 1. 文档目的
本文件用于区分“4 小时 V1 伪代码项目已完成”和“真实线上生产系统上线前仍需补齐”的内容，避免把设计稿误判为已完成生产实现。

## 2. 当前 V1 已具备
| 能力 | 状态 |
|---|---|
| 好友系统 | 已具备 `Friendship` 双向单向边与幂等 upsert |
| 发布动态 | 已具备 `Post + Outbox + Idempotency` 主写语义 |
| 查询个人动态 | 已具备 `author_id + created_at` cursor 分页 |
| 查询好友时间线 | 已具备 `FeedInbox -> BatchGet(Post)` 主链路 |
| 禁止读时聚合 | 已明确禁止拉取 5000 好友动态再排序 |
| cursor 分页 | 已覆盖 Post / FeedInbox / Friendship |
| fan-out 写扩散意识 | 已有容量公式、chunk、backpressure、dead-letter |
| 高压入口治理 | 已定义 `accepted / accepted_but_delayed / rejected_retry_later` 语义 |
| retention 边界 | 已定义近期时间线窗口与分区滚动清理 |
| 分片意识 | 已定义 route contract 与禁止全分片广播 |

## 3. 真实上线前必须补齐
### 3.1 容量与压测
- 回填 `publish_peak_qps`、`avg_friend_count`、`p95_friend_count`。
- 压测单 worker 的 `write_qps_per_worker`。
- 验证 `backlog_recovery_time` 是否满足目标。
- 验证首页 `BatchGet(Post)` 的 p95/p99。
- 验证 retention 后台清理不会影响在线读写。

### 3.2 存储与分区
- 确定真实数据库或存储系统。
- 确定 `Friendship`、`FeedInbox`、`Post`、`Outbox` 的实际分片数。
- 设计 route metadata 或 post_id 可路由规则。
- 设计热点分区识别与扩容策略。
- 验证批读/批写不会广播所有分片。

### 3.3 异步分发
- 确定真实队列或 outbox 表实现。
- 确定 worker 调度、并发上限和重试 backoff。
- 建立 dead-letter 处理流程。
- 建立 reconcile 扫描与补偿任务。
- 建立 fan-out lag 告警。

### 3.4 读路径治理
- 建立 first-page cache 的容量、TTL、命中率目标。
- 建立 cache failure fallback 验证。
- 建立 `BatchGet(Post)` 聚合批读层。
- 建立热点 Post 或热点 viewer 的限流策略。

### 3.5 数据生命周期
- 确定 retention 执行窗口。
- 确定每轮清理用户数与删除 chunk。
- 明确 retention window 外历史查询的产品语义。
- 如果需要历史查询，设计独立归档链路，不能回退到读时聚合。

### 3.6 可观测性
上线前必须具备以下最小指标：
- `publish_accept_latency_p99`
- `publish_admission_rejected_count`
- `publish_admission_delayed_count`
- `fanout_lag_p95`
- `fanout_lag_p99`
- `outbox_backlog_by_partition`
- `feedinbox_write_failure_rate`
- `timeline_read_latency_p99`
- `timeline_batch_get_post_latency_p99`
- `timeline_first_page_cache_hit_rate`
- `dead_letter_count`
- `retention_deleted_rows`
- `retention_paused_cycles`

## 4. 当前仍不承诺的内容
- 不承诺 retention window 外历史好友时间线完整可查。
- 不承诺所有好友立即可见，只承诺最终一致。
- 不承诺无压测即可支撑任意峰值。
- 不承诺大V/广播型用户已完成 push/pull hybrid。
- 不承诺具备完整生产级容灾、运维与自动扩缩容。

## 5. 最终上线门槛
只有同时满足以下条件，才可以从 V1 设计稿进入真实生产试运行：
- 容量公式已有真实压测参数回填。
- backlog 恢复时间小于目标窗口。
- 首页读取 p99 满足目标。
- retention 清理任务不会影响在线读写 p99。
- dead-letter 与 reconcile 有可执行处理流程。
- 所有 cache 失败均能 fallback 到正式链路。
- 所有批读/批写均按 route 分组，禁止全分片广播。
