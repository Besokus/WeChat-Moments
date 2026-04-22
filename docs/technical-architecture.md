# 朋友圈最小闭环项目技术文档（持续维护）

## 1. 文档定位
- 本文档是项目技术架构与选型依据的唯一汇总入口。
- 目标：在 4 小时内交付“可讲清楚、可复用、可持续维护”的最小朋友圈伪代码方案。
- 范围：只覆盖好友系统、发布动态、查询个人动态、查询好友时间线。

## 2. 约束与目标
### 2.1 业务目标
- 提供微信朋友圈最小闭环能力，不扩展到完整社交平台。

### 2.2 规模约束
- 用户量级：10M+
- 单用户好友上限：5000
- 平均每用户动态数：100

### 2.3 技术约束
- 仅输出伪代码与架构说明，不实现真实框架/数据库/中间件。
- 所有列表查询必须支持 `cursor + limit`。
- 禁止读时聚合 5000 好友全量动态后排序。
- 禁止全表扫描与深分页默认 `offset`。

## 3. 架构总览
### 3.1 分层
- `models`：数据模型与索引说明
- `repositories`：存储访问契约（接口级，不含具体 DB 实现）
- `services`：核心业务主流程
- `controllers`：请求参数进入 service 的薄层转发
- `flows`：端到端流程骨架（用于讲解/评审）
- `docs`：架构原则、交付标准、技术决策

### 3.2 核心模型
- `User`
- `Friendship`（双向关系采用两条单向边）
- `Post`
- `FeedInbox`（时间线轻量索引）

### 3.3 正式时间线路径
- 写路径：`publishMoment -> Post -> fan-out 写 FeedInbox`
- 读路径：`listFriendMoments -> FeedInbox 分页 -> batch 回查 Post`

## 4. 关键技术选型与取舍
### 4.1 时间线方案：`fan-out on write`
- 选择原因：
  - 在“5000 好友上限”下，写扩散成本可预期。
  - 将复杂度从读路径前移到写路径，读延迟更稳定。
- 不选原因（读时聚合）：
  - 潜在候选集可达 `5000 * 100 = 50万`。
  - 跨分区读取与排序开销高，尾延迟不可控。

### 4.2 分页方案：`cursor + limit`
- 选择原因：
  - 对大数据量和深分页更稳定。
  - 可配合索引 `(created_at, post_id)` 实现有序连续翻页。
- 不选原因（offset 深分页）：
  - 偏移量越大，扫描/跳过成本越高。

### 4.3 数据存储策略：FeedInbox 只存轻量引用
- 只存：`user_id, post_id, author_id, created_at`
- 不存：完整正文
- 原因：降低写扩散存储放大与一致性维护复杂度。

## 5. 模块职责与达标线
### 5.1 Friend 模块
- 职责：建立双向好友关系、按用户分页读取好友列表。
- 达标：`addFriend` 写两条单向边；`listFriends` 支持 `cursor + limit`。

### 5.2 Moment/Post 模块
- 职责：发布动态、按作者分页读取个人动态。
- 达标：`publishMoment` 完成写 Post 与 fan-out；`listUserMoments` 倒序分页。

### 5.3 Timeline 模块
- 职责：从 viewer 视角分页读取好友动态。
- 达标：必须走 `FeedInbox -> BatchGet(Post)`，并返回 `PageResult`。

## 6. 索引与分片建议（伪设计）
- `Friendship`: `UK(userId, friendId)` + `IDX(userId, createdAt DESC, friendId)`
- `Post`: `PK(postId)` + `IDX(authorId, createdAt DESC, postId)`
- `FeedInbox`: `PK(userId, createdAt DESC, postId)`（可选 `UK(userId, postId)`）
- 分片建议：
  - `Friendship`、`FeedInbox` 优先按 `userId` 路由
  - `Post` 按 `authorId` 或可路由主键策略

## 7. 风险与边界
- 本项目是伪代码架构交付，不等于生产可运行系统。
- 不覆盖评论、点赞、推荐流、审核、权限细分、多媒体。
- 对超高活跃用户（明星场景）仅给出演进方向，不在本期实现。
- 不引入控制器/框架/真实 SQL 实现；仅维护模型、仓储契约、服务流程与文档一致性。
- 时间线正式路径唯一：`FeedInbox -> BatchGet(Post)`；禁止回退到读时多好友动态归并。

## 8. 演进方向（仅记录，不实现）
- 异步 fan-out（队列化）
- 批量写入优化
- 热点用户 push/pull 混合策略
- Inbox 压缩与冷热分层

## 9. 持续维护机制
### 9.1 更新触发条件
- 任一核心模型字段变化（User/Friendship/Post/FeedInbox）
- 任一主流程变化（publish/listUser/listTimeline）
- 时间线策略变化（写扩散/读聚合）
- 分页与索引策略变化
- 交付边界变化（新增或移除能力）

### 9.2 更新要求
- 更新“架构总览”“技术选型与取舍”“模块达标线”中的对应章节。
- 在“变更记录”追加一条说明：改了什么、为什么改、影响范围。
- 若变更与最终交付标准冲突，必须同步更新 `docs/final-delivery-spec.md`。

## 10. 变更记录
- `2026-04-21`：初始化文档；确认正式时间线路径为 `FeedInbox + fan-out on write`；收敛接口与服务契约一致性。
- `2026-04-22`：补充高并发治理条款（分区消费/背压阈值/热点合并/retention）；新增时间线读放大优化（post_id 去重、缺失降级跳过）；同步固化技术边界（仅伪代码、禁止读扩散回退）。

## 11. 问题清单与优化实施计划（高并发/高负载）
### 11.1 P0 优化（必须完成）
- 问题：`publishMoment` 的 Post 与 FeedInbox 非原子边界。
- 实施：改为原子写 `Post + Outbox + Idempotency`，异步 worker 执行 fan-out。
- 当前状态：已落地到 `MomentService` 与 `Outbox` 契约。

- 问题：缺少请求级幂等，重试可能重复写。
- 实施：`addFriend/publishMoment` 强制 `request_id`，引入 `IdempotencyRepository`。
- 当前状态：已落地到 Friend/Moment API、Service、Repository 契约。

- 问题：同步 fan-out 写放大。
- 实施：`FanoutWorkerService` 异步消费 outbox，`BatchUpsert + chunk(200)`。
- 当前状态：已落地伪代码。

### 11.2 P1 优化（当前版本完成定义）
- 问题：分片只在文档层，契约未体现。
- 实施：仓储接口补充 `RouteContext(route_key)`。

- 问题：缺少失败恢复机制。
- 实施：新增 `ReconcileFeedInboxFlow`，对 FAILED/PENDING outbox 进行重放修复。

- 问题：随机读放大。
- 实施：保持 `FeedInbox -> BatchGet(Post)`，并规定“以 inbox 顺序组装返回”；后续可扩展批量局部性优化。

### 11.3 P2 优化（规则化）
- 明确游标并发语义：分页谓词固定为 `created_at < x OR (created_at = x AND post_id < y)`。
- 明确 FeedInbox 生命周期接口：`DeleteBefore` 归档/清理入口。
- 统一命名口径：保留 `Post` 作为数据实体，`Moment` 仅作为业务语义名。

### 11.4 执行优先级
1. 先做 P0：幂等、原子边界、异步 fan-out。
2. 再做 P1：分片路由契约、补偿流程。
3. 最后做 P2：游标语义、生命周期与命名规范固化。


## 12. 2026-04-22 高并发治理强约束（已落地到伪代码）
- Outbox 分区消费：按 partition 拉取事件，限制每分区每周期最大处理量。
- 背压阈值：PARTITION_PENDING_SOFT_LIMIT=20000 触发降速；PARTITION_PENDING_HARD_LIMIT=100000 触发告警并切到最小配额持续排空（避免 backlog 冻结）。
- 热点作者治理：worker 采用 author 短窗口合并（MERGE_WINDOW_SECONDS=5），单次好友读取复用到多个事件。
- 时间线读放大治理：请求内 post_id 去重，仓储内部强制按分片聚合批读；缺失 post 走降级跳过，不阻塞整页返回。
- 恢复 SLA：对账任务扫描周期 5 分钟，最大允许恢复窗口 15 分钟，超过重试上限进入 dead-letter。
- 数据生命周期硬规则：FeedInbox 执行 365 天保留策略，并强制每用户最大 20000 行上限（超出删除最老数据）。

## 13. 本轮修改完善（架构改进与边界优化）
### 13.1 架构改进
- 时间线读取链路完成稳定性增强：分页上限常量化、post_id 去重后批量回查、缺失 post 单条降级。
- 读路径继续保持轻量稳定：先读 `FeedInbox` 再批量回查 `Post`，不引入读时 fan-in 全量归并。
- 与高并发治理条款保持一致：优化点均落在既有仓储契约与服务伪代码边界内。

### 13.2 边界优化
- 技术交付边界进一步收敛为“伪代码 + 架构文档 + 约束说明”，不扩展运行时工程实现。
- 明确禁止项升级为维护规则：禁止全表扫描、禁止深分页 offset 默认方案、禁止时间线读扩散。
- 文档维护触发器生效：当模型/主流程/索引/边界变化时，必须同步更新本技术文档与变更记录。

## 13. 2026-04-22 最终缺口修复说明
- 幂等返回语义：publishMoment 的 IdempotencyRecord.result_ref 必须保存 post_id；幂等作用域必须为 op_name + actor_id + request_id，重复请求必须返回原始 post_id，禁止返回 0。
- fan-out 批量边界：FanoutWorker 按最终 FeedInbox item 数切分，公式为每批 items <= CHUNK_SIZE，避免 friend chunk * event count 放大。
- 原子边界前提：Post + Outbox + Idempotency 必须在同 route/partition 事务内提交；否则需改为主写成功后可靠补偿语义。
- 容量公式：required_fanout_write_capacity >= publish_qps * avg_friend_count；当实际能力低于该值时，系统必须承认 outbox 延迟会上升，并依赖背压阈值保护写链路。
- 默认恢复目标：max_fanout_lag_target = 300s，max_backlog_recovery_time = 1800s（默认值，需压测校准）。
- FeedInbox 历史边界：V1 只保证 retention 窗口内的近期好友时间线；超过 retention 的历史归档查询不属于当前最小闭环。
- 好友修复语义：EnqueueEdgePairRepair 是持久化补偿任务入口，由 RepairFriendshipEdgePairFlow 异步幂等补齐双向边。
- 分页语义：Post/FeedInbox cursor 使用 created_at + post_id；Friendship cursor 使用 created_at + friend_id。

