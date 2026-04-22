# 朋友圈最小闭环 V1 实现说明（最终记录）

## 1. 文档目的
本文件用于固定当前 V1 版本的完整实现状态，作为：
- 课程作业提交说明
- 面试讲解主稿
- 后续迭代基线

本文件只描述当前已实现的伪代码与设计，不引入新功能。

## 2. 题目目标与完成范围
在 4 小时内完成“微信朋友圈最小闭环”伪代码实现，覆盖：
1. 好友系统
2. 发布动态
3. 查询某个人的动态列表
4. 从某个人视角查询其所有好友动态时间线

当前 V1 已完成以上 4 项能力。

## 3. 规模约束与设计影响
硬约束：
- 用户量级：10M+
- 每人最多好友数：5000
- 平均每人动态数：100

对设计的直接影响：
- 时间线不能走读时聚合所有好友动态，必须走预计算索引（FeedInbox）。
- 发布链路必须承认写扩散代价，并提供分区消费、背压、重试语义。
- 列表查询必须是 cursor 分页，不使用深 offset。
- 存储访问必须具备 user/author 维度路由与索引意识，避免全表扫描。

## 4. 架构选型（V1 固定）
核心模型：
- User
- Friendship
- Post
- FeedInbox

时间线策略：
- fan-out on write（写扩散）

正式主链路：
- 写：publishMoment -> Post + Outbox + Idempotency -> 异步 fan-out -> FeedInbox
- 读：FeedInbox 分页 -> BatchGet(Post) -> 组装返回

禁止方案：
- 读时按好友集合拉取并全量排序
- 全表扫描
- 深分页 offset 默认方案

## 5. 目录与模块落盘
代码结构（伪代码）：
- `domain/`：服务与模型
- `storage/`：仓储契约、分页与 ID 契约
- `flows/`：端到端流程骨架
- `interfaces/`：接口定义
- `controllers/`：入口转发骨架
- `docs/`：架构与交付文档

V1 关键文件：
- `domain/models.pseudo`
- `domain/friend_service.pseudo`
- `domain/moment_service.pseudo`
- `domain/timeline_service.pseudo`
- `domain/fanout_worker_service.pseudo`
- `storage/repositories.pseudo`
- `storage/cursor.pseudo`
- `storage/id_generator.pseudo`
- `flows/publish_moment_flow.pseudo`
- `flows/reconcile_feed_inbox_flow.pseudo`
- `flows/feed_retention_flow.pseudo`
- `flows/repair_friendship_flow.pseudo`

## 6. 核心数据模型（V1）
### 6.1 User
作用：用户身份与基础合法性校验。  
核心字段：`user_id`, `status`, `created_at`。  
访问模式：按 `user_id` 点查。

### 6.2 Friendship
作用：好友关系表达，采用双向两条单向边。  
核心字段：`user_id`, `friend_id`, `status`, `created_at`。  
访问模式：按 `user_id` 分页查询好友列表。  
约束语义：唯一键 `(user_id, friend_id)`，支持幂等 upsert。

### 6.3 Post
作用：动态事实数据源。  
核心字段：`post_id`, `author_id`, `content_text`, `created_at`。  
访问模式：按 `author_id + created_at DESC` 查询个人动态，按 `post_id` 批量回查。

### 6.4 FeedInbox
作用：好友时间线读取索引（轻量引用，不存完整正文）。  
核心字段：`user_id`, `post_id`, `author_id`, `created_at`。  
访问模式：按 `user_id + created_at DESC` 分页读取，再批量回查 Post。

## 7. 核心接口与服务职责（V1）
### 7.1 好友
- `addFriend(userId, friendId, requestId)`
  - 请求级幂等
  - 双边 upsert
  - 失败进入持久化补偿任务
- `listFriends(userId, cursor, limit)`
  - 基于 Friendship 的 cursor 分页

### 7.2 动态
- `publishMoment(userId, content, requestId)`
  - 原子写 `Post + Outbox + Idempotency`
  - duplicate request 返回原始 `post_id`
  - 异步 fan-out
- `listUserMoments(authorId, cursor, limit)`
  - 作者维度倒序分页

### 7.3 时间线
- `listFriendMoments(viewerId, cursor, limit)`
  - FeedInbox 主链路读取
  - post_id 去重后批量回查
  - 缺失 post 时降级补页

## 8. 核心流程（V1）
### 8.1 好友关系建立
1. 参数校验 + request_id 幂等判断  
2. 双向边写入（A->B, B->A）  
3. 失败时落持久化补偿任务，异步补齐双向边

### 8.2 发布动态
1. 参数校验 + request_id 幂等判断  
2. 生成 `post_id` 与 `outbox_event_id`  
3. 原子写入 Post、Outbox、Idempotency(result_ref=post_id)  
4. 返回 accepted + post_id  
5. 异步 worker 消费 outbox 执行 fan-out

### 8.3 查询个人动态
1. 按 `author_id` 路由  
2. 索引倒序拉取 `size+1`  
3. 生成 `next_cursor` 与 `has_more`

### 8.4 查询好友时间线
1. 按 viewer 读取 FeedInbox（cursor）  
2. post_id 去重批量回查 Post  
3. 缺失 post 跳过并继续补页  
4. 返回 PageResult

## 9. 关键工程语义（V1）
### 9.1 幂等语义
- `publishMoment`：幂等键作用域为 `op_name + actor_id + request_id`，重复请求必须返回同一 `post_id`（来自 `IdempotencyRecord.result_ref`）。
- `addFriend`：幂等键作用域为 `op_name + actor_id + request_id`，重复请求不重复建边。

### 9.2 一致性与恢复
- 发布主写边界：`Post + Outbox + Idempotency` 原子化（前提：同 route/partition 事务边界）。
- fan-out 失败：重试 + backoff + dead-letter。
- 对账修复：`ReconcileFeedInboxFlow` 扫描并重放缺失项。
- 好友边修复：`RepairFriendshipEdgePairFlow` 幂等补齐双边关系。

### 9.3 分页语义
- Post/FeedInbox cursor：`created_at + post_id`
- Friendship cursor：`created_at + friend_id`
- 均使用 cursor，禁止深 offset。

### 9.4 fan-out 治理语义
- Outbox 分区消费
- 积压软硬阈值背压（硬降级仍保留最小消费配额排空）
- 热点 author 短窗口合并
- 批写按最终 FeedInbox item 数切分（`items <= CHUNK_SIZE`）

### 9.5 时间线补页边界
- 读取时允许 overfetch，补页上限由 `MAX_INBOX_SCAN_FACTOR` 控制。

### 9.6 retention 边界
- V1 仅保证 retention window 内的近期时间线。
- retention window 外历史查询不在 V1 保证范围。

## 10. 索引、路由与容量意识
### 10.1 索引建议
- Friendship：`UK(user_id, friend_id)` + `IDX(user_id, created_at DESC, friend_id)`
- Post：`PK(post_id)` + `IDX(author_id, created_at DESC, post_id)`
- FeedInbox：`PK(user_id, created_at DESC, post_id)`（可选 `UK(user_id, post_id)`）

### 10.2 路由意识
- `Friendship`、`FeedInbox` 优先按 `user_id` 路由。
- `Post` 通过 `author_id` 或可路由主键策略。
- `post_id` 必须全局唯一且可路由到 Post 分片。

### 10.3 容量公式（V1）
- `required_fanout_write_capacity >= publish_qps * avg_friend_count`
- 当实际能力低于该值时，outbox 延迟会上升，由背压阈值保护链路。

- `max_fanout_lag_target = 300s`（默认目标值）
- `max_backlog_recovery_time = 1800s`（默认目标值）
说明：阈值参数为 V1 默认值，后续需要通过压测回填校准。

### 10.4 未来演进（仅说明）
当前 V1 在题目约束下统一采用 `fan-out on write` 是合理的：好友上限为 5000，写扩散成本可预期，且可换取更稳定的时间线读取路径（`FeedInbox -> BatchGet(Post)`）。  
当未来出现高活跃用户/广播型用户/大V时，持续高频发布会放大 fan-out 写压力，并推高 outbox 积压与分区热点风险。  
后续可在不改变主模型（`Friendship + Post + FeedInbox`）前提下，演进为 push/pull hybrid：普通用户继续 push，大V按 pull 路径供粉丝侧读取，以降低极端写放大。  
该演进不属于当前 V1 实现范围，仅作为容量扩展方向保留。
## 11. 边界与非目标（V1）
不做：
- 评论、点赞、媒体、推荐流、审核
- 真实中间件接入与生产部署细节
- 完整生产级高可用体系

保持：
- 只交付伪代码 + 架构文档
- 不扩展题目范围
- 维持 4 小时快速项目可讲清、可提交、可答辩

## 12. 当前 V1 验收结论
基于现有伪代码与文档，当前 V1 已满足本题交付标准：
- 覆盖四个核心功能
- 架构主链路清晰且与约束匹配
- 具备分页、索引、分区、写扩散、幂等、补偿、retention 边界意识
- 保持最小闭环，不发生范围失控

本版本可作为“最终提交版基础稿”用于课程作业与面试讲解。


