# 容量规划与压测验收方案

## 1. 文档目的
本文件用于把朋友圈 V1 的高并发风险转化为可计算、可压测、可验收的容量口径。它不是生产部署配置，也不引入新业务能力。

## 2. 适用范围
- 发布动态入口容量
- Outbox 与 fan-out worker 分发容量
- FeedInbox 写入容量
- 首页时间线读取容量
- FeedInbox retention 后台清理容量

不覆盖评论、点赞、媒体、推荐流、完整生产部署与真实中间件选型。

## 3. 核心容量变量
| 变量 | 含义 |
|---|---|
| `publish_peak_qps` | 发布动态入口峰值 QPS |
| `avg_friend_count` | 发布者平均好友数 |
| `p95_friend_count` | 发布者 P95 好友数 |
| `max_friend_count` | 单用户好友上限，当前题目固定为 5000 |
| `partition_count` | outbox / FeedInbox 写入分区数 |
| `worker_per_partition` | 每个分区的 worker 并发数 |
| `write_qps_per_worker` | 单 worker 可稳定写入的 FeedInbox QPS |
| `backlog_size` | 当前 outbox 积压事件数 |
| `target_fanout_lag_seconds` | 允许的 fan-out 可见性延迟目标 |
| `target_backlog_recovery_seconds` | backlog 恢复目标 |

## 4. fan-out 写扩散预算
单条动态的最坏写扩散：

```text
max_fanout_writes_per_post = 5000
```

常态写扩散速率：

```text
incoming_fanout_write_qps = publish_peak_qps * avg_friend_count
```

保守容量评估：

```text
p95_fanout_write_qps = publish_peak_qps * p95_friend_count
worst_case_fanout_write_qps = publish_peak_qps * max_friend_count
```

worker 写入能力：

```text
worker_write_capacity = partition_count * worker_per_partition * write_qps_per_worker
```

backlog 增长：

```text
backlog_growth_qps = max(0, incoming_fanout_write_qps - worker_write_capacity)
```

backlog 恢复时间：

```text
backlog_recovery_time = backlog_size / max(1, worker_write_capacity - incoming_fanout_write_qps)
```

验收标准：当 `worker_write_capacity < incoming_fanout_write_qps` 时，系统必须进入 backlog/降级状态，不能宣称实时 fan-out。

## 5. 发布入口准入控制
仅靠异步 fan-out 会把压力后移到 outbox。真实高压下，发布入口必须有 admission control。

| 状态 | 条件 | 行为 |
|---|---|---|
| `ACCEPTED` | backlog 低于软阈值 | 正常接收，返回 `accepted` |
| `ACCEPTED_BUT_DELAYED` | backlog 超过软阈值 | 接收主写，明确时间线延迟可见 |
| `REJECTED_RETRY_LATER` | backlog 超过硬阈值 | 受控拒绝，请客户端退避重试 |

验收标准：入口不能在 backlog 已失控时继续无限返回普通 `accepted`。

## 6. 首页读取容量
正式路径仍然是：

```text
FeedInbox.ListByUser -> BatchGet(Post) -> assemble PageResult
```

首页读峰值的主要瓶颈不是 FeedInbox 顺序读，而是 `BatchGet(Post)` 的跨分片回表与随机读放大。

建议观测指标：
- `timeline_read_qps`
- `timeline_read_latency_p95`
- `timeline_read_latency_p99`
- `timeline_batch_get_post_count`
- `timeline_batch_get_post_latency_p95`
- `timeline_first_page_cache_hit_rate`

验收标准：首屏 cache 失败时必须回到正式路径；无 cache 时系统仍成立，但高峰延迟会上升。

## 7. FeedInbox retention 容量
retention 是 V1 的近期时间线边界，不是完整历史归档方案。

清理容量必须受控：

```text
retention_users_per_cycle <= MAX_USERS_PER_PARTITION_CYCLE
retention_delete_rows_per_user_batch <= DELETE_CHUNK_SIZE
retention_task_priority = low
```

执行要求：
- 按 `FeedInbox` route partition 滚动执行。
- 禁止全局全表扫描。
- 在线读写延迟升高时暂停或降速。
- 超过 retention window 的历史时间线不属于 V1 保证范围。

## 8. 压测场景
| 场景 | 压测目标 |
|---|---|
| 5000 好友用户发布 1 条动态 | 验证单事件最多 5000 inbox 写入是否进入 outbox 并可恢复 |
| 1000 用户同时发布 | 验证 backlog 增长速率与恢复时间 |
| 大量用户刷首页 | 验证 `BatchGet(Post)` 延迟与首屏 cache 命中率 |
| 单用户深翻页 | 验证 cursor 查询不退化为 offset 深分页 |
| retention 后台运行 | 验证清理任务不影响在线读写 p99 |

## 9. V1 通过标准
- 能用公式解释 fan-out 写扩散压力。
- 能说明 worker 欠配时 backlog 必然增长。
- 能通过 admission control 避免入口无限接单。
- 能定位首页读峰值瓶颈在 `BatchGet(Post)`。
- 能说明 retention 是近期时间线边界，而非完整历史方案。
