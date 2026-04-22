# 失败场景矩阵与恢复策略

## 1. 文档目的
本文件用于明确朋友圈 V1 在高并发、高负载、大规模约束下的失败边界与恢复策略，避免只写“重试/补偿”但没有场景对应关系。

## 2. 设计原则
- 发布主写优先保证 `Post + Outbox + Idempotency` 原子边界。
- fan-out 允许最终一致，不承诺所有好友立即可见。
- FeedInbox 写入必须幂等，重复消费不能产生重复时间线项。
- 查询链路必须可降级返回，不能因单条缺失 Post 让整页失败。
- retention window 外历史时间线不属于 V1 保证范围。

## 3. 失败矩阵
| 场景 | 影响 | V1 处理策略 | 验收标准 |
|---|---|---|---|
| 用户参数非法 | 请求无效 | 直接拒绝 | 不写 Post / Outbox |
| `request_id` 缺失 | 无法幂等 | 直接拒绝 | 不执行主写 |
| 重复 publish 请求 | 客户端重试 | 返回原始 `post_id` | 不重复创建 Post |
| Post 写失败 | 发布失败 | 返回失败 | 不生成 outbox |
| Outbox 写失败 | 分发不可恢复 | 必须与 Post 同原子边界 | 禁止出现 Post 成功但无 Outbox |
| Idempotency 写失败 | 重试语义不闭合 | 必须与 Post/Outbox 同原子边界 | 禁止重复请求生成新 Post |
| fan-out 部分失败 | 部分好友不可见 | Outbox 重试 + 幂等 upsert | 最终补齐 FeedInbox |
| fan-out 重复消费 | 可能重复时间线 | `UK(user_id, post_id)` / upsert | 同一用户不出现重复 post |
| backlog 超过软阈值 | 可见性延迟上升 | 返回 `accepted_but_delayed` | 不承诺实时可见 |
| backlog 超过硬阈值 | 系统过载 | 返回 `rejected_retry_later` | 入口不无限接单 |
| dead-letter | 自动重试失败 | 记录人工/离线修复入口 | 不静默丢失 |
| Post 回表缺失 | 时间线项不可组装 | 跳过并 overfetch 补页 | 整页不失败 |
| FeedInbox 超过 retention | 历史不可查 | V1 明确不保证 | 不回退到读时聚合 |
| Friendship 单边 | 好友关系不一致 | 持久化修复任务补齐双边 | 修复可重试幂等 |
| first-page cache 失败 | 读优化失效 | fallback 到正式链路 | cache 不是唯一真相源 |
| retention 清理压力过高 | 影响在线读写 | 低优先级暂停/降速 | 不压垮在线路径 |

## 4. 发布链路失败闭环
发布链路必须满足：

```text
Post + Outbox + Idempotency = 同 route/partition 原子提交
```

如果真实实现无法满足该前提，必须降级为可靠补偿语义，并明确：
- Post 成功但 Outbox 缺失如何发现。
- Idempotency 成功但 Post 缺失如何处理。
- 补偿扫描周期与最大恢复窗口。

## 5. fan-out 失败闭环
fan-out worker 必须满足：
- 按 outbox partition 消费。
- 按最终 `FeedInbox` item 数 chunk。
- 写入 FeedInbox 使用幂等 upsert。
- 失败重试使用 backoff。
- 超过重试上限进入 dead-letter。
- reconcile 能重放未完成分发。

## 6. 查询链路失败闭环
好友时间线查询必须满足：
- 正式路径唯一：`FeedInbox -> BatchGet(Post)`。
- cache miss / cache error 不影响正式路径。
- `BatchGet(Post)` 缺失单条时跳过，不让整页失败。
- overfetch 受 `MAX_INBOX_SCAN_FACTOR` 控制。
- 返回条数允许小于 limit。

## 7. V1 边界
V1 不是完整生产级容灾系统。它只要求在伪代码和架构说明中把失败分类、恢复路径和不可保证边界讲清楚。
