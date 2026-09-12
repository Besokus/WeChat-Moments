# 微信朋友圈最小闭环 V1

## 1. 这是什么项目
这是一个“微信朋友圈最小闭环”的伪代码项目，目标不是做完整社交平台，而是在 **4 小时** 的面试/作业窗口里，交付一套真正能讲清楚、能落文档、能体现大规模约束意识的方案。

它只做四件事：
- 好友系统
- 发布动态
- 查询某个人的动态列表
- 从某个人的视角查询其所有好友的动态时间线

这个项目的价值不在“功能多”，而在于它把朋友圈里最难的那部分讲透了：
- 高并发下如何写
- 大规模下如何读
- 发生故障时如何恢复
- 压力过高时如何有尊严地退化
- 为什么这套设计在 `10M+ / 5000 好友 / 100 动态` 约束下成立

## 2. 题目约束，为什么它们是真约束
当前默认约束是：
- 用户量级：`10M+`
- 单用户最多好友数：`5000`
- 平均每用户动态数：`100`

这三个数不是装饰，它们直接改变架构选择：
- `5000` 好友意味着不能在读路径临时聚合所有好友动态再排序
- `100` 动态/人意味着 Post 和 FeedInbox 的总体数据规模不能小
- `10M+` 用户意味着所有主路径都必须是索引友好的、分片友好的、禁止全表扫描的

所以，这不是“朋友圈小项目”，而是“在强约束下做最小闭环”。

## 3. 主架构结论
当前 V1 的主架构已经固定为：
- `Friendship`
- `Post`
- `FeedInbox`

时间线策略固定为：
- `fan-out on write`

正式主链路固定为：
- 写：`publishMoment -> Post + Outbox + Idempotency -> 异步 fan-out -> FeedInbox`
- 读：`FeedInbox 分页 -> BatchGet(Post) -> 组装返回`

这代表几件关键事：
- 好友时间线不是读时临时归并出来的
- 发布请求不是同步等待所有好友可见
- 高压场景下不是“无限 accepted”，而是允许延迟可见甚至受控拒绝
- `FeedInbox` 不是冗余表，而是时间线的预计算索引

## 4. 为什么这套架构是对的
### 4.1 为什么不用读时聚合
如果一个用户最多 5000 个好友，平均每个好友 100 条动态，那么单次时间线请求潜在候选集理论上可以到 `50 万` 量级。读时去好友集合里拉数据、再排序、再分页，会把问题放大成：
- 跨分区读取
- 高 IO / 高网络开销
- 应用层排序和归并成本高
- 深分页越翻越慢

所以这条路不是本题的主路径。

### 4.2 为什么用 `FeedInbox`
`FeedInbox` 把“好友时间线”前置成写扩散问题，读的时候就只做：
- 顺序分页读 inbox
- 批量回查 Post
- 组装返回

这对用户体验的意义很直接：
- 首页读延迟更稳定
- 读路径更可预测
- 适合大规模并发访问

### 4.3 为什么用 `fan-out on write`
写扩散的代价是明确的，但它是可预算的。因为好友数上限只有 5000，所以“最坏情况会写多少”是能估的，而不是无限展开的。对本题来说，读路径稳定比写路径轻更重要。

## 5. 目录结构
项目按职责拆成以下目录：

- `domain/`
  - 核心服务和领域伪代码
- `storage/`
  - 仓储契约、分页、ID、路由约束
- `flows/`
  - 端到端流程骨架
- `interfaces/`
  - 接口定义
- `controllers/`
  - 请求入口薄层转发
- `docs/`
  - 架构说明、交付边界、工程化增强说明

关键文件：
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

## 6. 数据模型，为什么这样拆
### 6.1 User
作用：
- 最小身份模型
- 用于用户存在性校验

核心字段：
- `user_id`
- `status`
- `created_at`

访问方式：
- 按 `user_id` 点查

### 6.2 Friendship
作用：
- 表示好友关系
- 使用两条单向边表示双向好友

核心字段：
- `user_id`
- `friend_id`
- `status`
- `created_at`

访问方式：
- 按 `user_id` 分页读取好友列表

为什么这样做：
- 易于按用户维度路由
- 易于幂等 upsert
- 避免图遍历型查询

### 6.3 Post
作用：
- 动态事实源
- 真正保存内容正文

核心字段：
- `post_id`
- `author_id`
- `content_text`
- `created_at`

访问方式：
- `author_id + created_at DESC` 查询个人动态
- `post_id` 批量回查内容

### 6.4 FeedInbox
作用：
- 好友时间线的读取索引
- 只保存轻量引用，不保存完整正文

核心字段：
- `user_id`
- `post_id`
- `author_id`
- `created_at`

访问方式：
- 按 `user_id + created_at DESC` 分页读取
- 再批量回查 `Post`

## 7. 核心服务，职责边界很清楚
### 7.1 FriendService
- `addFriend(userId, friendId, requestId)`
  - 请求级幂等
  - 双边 upsert
  - 失败进入持久化补偿任务
- `listFriends(userId, cursor, limit)`
  - 基于 `Friendship` cursor 分页

### 7.2 MomentService
- `publishMoment(userId, content, requestId)`
  - 原子写 `Post + Outbox + Idempotency`
  - duplicate request 返回原始 `post_id`
  - 入口支持 admission control
- `listUserMoments(authorId, cursor, limit)`
  - 作者维度倒序分页

### 7.3 TimelineService
- `listFriendMoments(viewerId, cursor, limit)`
  - `FeedInbox` 主链路读取
  - `post_id` 去重后批量回查
  - 缺失 `Post` 时降级补页

### 7.4 FanoutWorkerService
- 异步消费 outbox
- 按分区处理
- 软硬阈值背压
- 热点作者短窗口合并
- 批写按最终 `FeedInbox item` 数切分

这套拆法的好处是：
- 主写、扩散、查询、恢复四条链路分离
- 每个模块的失败边界和职责边界都可以单独讲清楚
- 不需要把所有逻辑揉成一个“大服务”

## 8. 核心流程，哪些地方是真正的工程亮点
### 8.1 好友关系建立
1. 参数校验
2. request_id 幂等判断
3. 双向边写入
4. 失败时落持久化补偿任务

亮点：
- 不把好友关系写成“看起来简单、实际不稳”的单步操作
- 幂等与补偿让关系建立可重试、可恢复

### 8.2 发布动态
1. 参数校验
2. request_id 幂等判断
3. 入口 admission control
4. 原子写入 `Post + Outbox + Idempotency`
5. 返回 `accepted` / `accepted_but_delayed` / `rejected_retry_later`
6. 异步 worker 执行 fan-out

亮点：
- 发布链路不是“发出去就完了”，而是有流量治理闭环
- 高压时系统会主动退化，而不是无限接单把自己拖死

### 8.3 查询个人动态
1. 按 `author_id` 路由
2. 索引倒序拉取
3. `cursor + limit` 返回

亮点：
- 路径单纯，索引明确，不引入多余复杂度

### 8.4 查询好友时间线
1. 按 viewer 读取 `FeedInbox`
2. `post_id` 去重
3. 批量回查 `Post`
4. 缺失项降级补页

亮点：
- 不因为单条 Post 缺失而让整页失败
- 支持 overfetch 和补页，避免“内容少一点就整体崩掉”

## 9. 并发设计，为什么它像一个真实系统
### 9.1 写扩散不是问题被忽略，而是被显式承认
`5000` 好友意味着单次发布最坏情况会产生大量 `FeedInbox` 写入。这里不是假装没有成本，而是把成本显式化：
- `outbox` 承担异步分发
- worker 承担并发消费
- chunk 承担批写边界
- backpressure 承担过载退让

### 9.2 admission control 是入口护栏
只靠消费端背压不够。因为如果入口一直接受请求，系统会从“短暂延迟”退化成“长时间积压”。所以 publish 入口必须能：
- 正常接收
- 延迟接收
- 受控拒绝

这就是为什么 `accepted_but_delayed` 和 `rejected_retry_later` 不是花活，而是真实高压系统里必须有的护栏。

### 9.3 分区消费不是装饰，是容量治理
Outbox / FeedInbox / Friendship / Post 都必须有路由意识。原因很简单：
- 不分区就会广播
- 广播就会拖垮系统
- 拖垮后你再做重试只是在放大失败

### 9.4 热点作者要特殊处理
高活跃用户/大V场景会把 fan-out 压力集中放大，所以 worker 需要：
- 短窗口合并
- 降速消费
- 最小消费配额
- dead-letter + reconcile

这说明这个方案不是“只会处理平均流量”，而是知道怎么面对热点。

## 10. 一致性，系统为什么能恢复
### 10.1 幂等
- `publishMoment` 幂等键：`op_name + actor_id + request_id`
- `addFriend` 幂等键：`op_name + actor_id + request_id`

这意味着：
- 重试不会重复发帖
- 重试不会重复加好友
- 请求可以安全重复提交

### 10.2 原子边界
`Post + Outbox + Idempotency` 需要在同 route/partition 事务边界内提交。这个前提不是装饰，而是保证“主写成功但扩散丢失”不会变成不可恢复漏洞。

### 10.3 最终一致
- fan-out 失败后重试、backoff、dead-letter
- `ReconcileFeedInboxFlow` 对账补偿
- `RepairFriendshipEdgePairFlow` 双边好友修复

这套逻辑的重点不是“绝对不失败”，而是“失败后能被发现、能被重放、能最终补齐”。

## 11. 降级设计，为什么不是“系统变差了”
高负载下的正确目标不是“所有请求都保持实时”，而是：
- 发布成功优先
- 时间线可短暂延迟
- 必要时受控拒绝
- 最终一致性仍保留

对应的对外语义非常明确：
- `accepted`
- `accepted_but_delayed`
- `rejected_retry_later`

这套降级逻辑的亮点在于：
- 不把压力无限堆给后端异步消费者
- 不把问题掩盖成“总会最终补好”
- 让系统在真实压力下能有明确、可解释、可恢复的退化路径

## 12. 边界与非目标，项目没有装作自己无所不能
V1 明确不做：
- 评论、点赞、媒体、推荐流、审核
- 真实中间件接入与生产部署细节
- 完整生产级高可用体系

V1 明确只保证：
- 近期时间线可用
- retention window 内数据可查
- 高压下行为可解释
- 失败后路径可恢复

这很重要，因为真正成熟的设计不是把所有东西都做了，而是把“我做什么、我不做什么、为什么”说清楚。

## 13. 这些亮点为什么能显示真实落地能力
这个项目不是纯文档堆砌，它的亮点是每个难点都有对应语义：

- `FeedInbox` 说明时间线是预计算索引，不是临时聚合
- `Outbox` 说明写扩散是异步分发，不是同步拖死入口
- `Idempotency` 说明重试是安全的，不会重复写
- `Reconcile` 说明失败是可修复的，不是静默丢失
- `admission control` 说明系统高压时会主动降级，不会无限接单
- `retention` 说明历史边界是被定义过的，不是假装无限历史可查

这些东西放在一起，才像一个真实系统，而不是一个只会写“伪代码”的答案。

## 14. 生产就绪补强，当前已经把边界讲明白了
为了避免把设计稿误判成生产实现，项目还补了三份生产就绪说明：
- `docs/capacity-plan.md`
- `docs/failure-matrix.md`
- `docs/production-readiness-checklist.md`

它们分别回答三个问题：
- 容量怎么算
- 失败怎么恢复
- 真上生产前还缺什么

这三份文档不改主架构，但把真实线上要面对的问题讲透了。

## 15. 怎么讲这份项目最合适
如果你拿这份项目去面试，可以按这个顺序讲：
1. 先讲约束
2. 再讲为什么不能读时聚合
3. 再讲为什么选 `Friendship + Post + FeedInbox`
4. 再讲 fan-out on write
5. 再讲 admission control、幂等、补偿、retention
6. 再讲为什么这套东西能在高压下落地
7. 最后讲未来演进到 push/pull hybrid

## 16. 对应文档索引
- [docs/v1-overview.md](docs/v1-overview.md)
- [docs/final-delivery-spec.md](docs/final-delivery-spec.md)
- [docs/technical-architecture.md](docs/technical-architecture.md)
- [docs/v1-enhancement-notes.md](docs/v1-enhancement-notes.md)
- [docs/capacity-plan.md](docs/capacity-plan.md)
- [docs/failure-matrix.md](docs/failure-matrix.md)
- [docs/production-readiness-checklist.md](docs/production-readiness-checklist.md)

## 17. 最后总结
这份 V1 的目标不是“朋友圈功能全做完”，而是把最小闭环、规模约束、并发治理、一致性、降级、恢复这些真实工程问题一次性讲明白。

它真正有价值的地方是：
- 主链路清晰
- 并发治理明确
- 失败恢复明确
- 降级语义明确
- 边界也明确

换句话说，这不是一个“看起来像系统设计”的答案，而是一个能对真实高压场景给出明确方案的答案。

## 18. V2 迭代版本（从 V1 演进）
如果需要展示下一版演进，可以直接阅读 `docs/v2-iteration.md`。它保留 V1 主架构不变，只把高活跃用户治理、push/pull hybrid、首屏缓存刚需化、分层 admission control 和更细的恢复优先级作为下一阶段演进方向。

V2 的核心不是重写，而是让系统更像真实线上：普通用户继续走 push，高活跃用户逐步走 pull/hybrid；读热点通过 cache 与 batch get 分层治理；写热点通过更细粒度 admission control 和恢复策略分层治理。
