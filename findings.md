# Findings

## Project Context
- 仓库当前为伪代码结构化目录：`docs/`, `interfaces/`, `domain/`, `storage/`, `flows/`
- 已有 `AGENTS.md` 约束，强调最小范围、可解释性、cursor 分页、禁止错误方案。

## Current Turn Requirements
- 仅输出 5 部分：
  1. 需求理解
  2. 核心架构选型
  3. 架构适配规模原因
  4. 核心数据流（发动态/查个人动态/查好友时间线）
  5. 项目边界（做什么/不做什么）
- 不直接输出大段伪代码

## Architectural Defaults (User-mandated)
- 数据模型：Friendship + Post + FeedInbox
- 时间线：fan-out on write
- 可简要提演进为 push/pull 混合，但本次不实现

## 2026-04-21 模型落盘更新
- 已将核心模型更新为 User/Friendship/Post/FeedInbox/PageResult。
- Friendship 明确为双向两条单向边表达。
- Post 查询模式明确 author_id + created_at DESC。
- FeedInbox 查询模式明确 user_id + created_at DESC，且仅存引用与轻量信息。
- 明确说明禁止读时动态聚合最多 5000 好友全量动态。

## 
2026-04-21 22:25:25
 核心功能伪代码落盘
- 已将 5 个核心函数落盘到 domain service 层。
- 文件：domain/friend_service.pseudo, domain/moment_service.pseudo, domain/feed_service.pseudo。
- publishPost 体现写 Post + 查好友 + 扇出写 FeedInbox，并注明异步 fan-out 建议。
- listUserPosts 使用 author 维度倒序 + cursor 分页。
- listFriendTimeline 使用 FeedInbox 游标分页 + 批量回查 Post，并明确禁止读时遍历 5000 好友归并。

## 2026-04-21 可交付版本说明固化
- 新增 docs/final-delivery-spec.md，明确项目成功标准与模块验收口径。
- 重点固化了 10M+/5000/100 约束如何落到模型、访问模式与分页策略。
- 明确读时不能动态聚合最多 5000 好友全量动态的工程原因。

## 2026-04-21 Repository层更新
- 已将 storage/repositories.pseudo 收敛为 FriendRepository 与 MomentRepository 两个接口。
- 所有列表查询方法显式包含 cursor 与 limit。
- 已补充按 authorId 批量查询最近动态能力：ListRecentByAuthors(author_ids, cursor, limit)。

## 2026-04-21 Service层重写
- 已按要求生成 FriendService、MomentService、TimelineService 主流程伪代码。
- 每个方法前均添加一句职责说明。

- listFriendMoments(viewerId, cursor, limit) 明确包含：查好友、提取friendIds、分页拉取最近动态、按createdAt倒序、返回PageResult。

## 2026-04-21 Controller层生成
- 新增 FriendController、MomentController、TimelineController 三个伪代码控制器。
- 控制器仅体现请求参数提取与 service 调用，不包含 HTTP 框架细节。
- 命名统一：addFriend/listFriends、publishMoment/listUserMoments、listFriendMoments。

## 2026-04-21 时间线架构收敛修复
- 结论：不需要同时保留 FeedInbox 写扩散与 friendIds 读时聚合两套正式路径。
- 原因：在 10M+ 用户、最多 5000 好友、人均 100 动态约束下，读时聚合会带来跨索引读取、排序和尾延迟风险。
- 修复：正式路径统一为 publishMoment 写 Post 后 fan-out 写 FeedInbox；listFriendMoments 从 FeedInbox 分页读取并批量回查 Post。
- 已移除 MomentRepository.ListRecentByAuthors 作为正式仓储能力，避免误用为主路径。

## 2026-04-21 接口契约一致性修复
- FriendAPI 改为与 FriendService 对齐：AddFriend 返回 Result，ListFriends 使用 cursor+limit 并返回 PageResult<Friendship>。
- MomentAPI 改为与 MomentService 对齐：PublishMoment 返回 PublishResult，ListUserMoments 返回 PageResult<Post>。
- TimelineAPI 改为与 TimelineService 对齐：ListFriendMoments 使用 cursor+limit 并返回 PageResult<Post>。

## 2026-04-21 技术文档主文件建立
- 新增 docs/technical-architecture.md，集中维护技术架构、选型依据、取舍和演进说明。
- 明确了 10M+/5000/100 约束下的主路径：FeedInbox + fan-out on write。
- 新增持续维护机制：触发条件、更新要求、变更记录规范。

## 2026-04-22 高并发稳定性修复落地
- 已将 P0/P1/P2 问题转为可执行实施计划并落地到伪代码。
- P0：publishMoment 改为 Post+Outbox+Idempotency 原子主写；fan-out 改为异步 worker 分批幂等写入。
- P0：addFriend / publishMoment 新增 request_id 幂等语义。
- P1：仓储契约补充 RouteContext 分片路由；新增 ReconcileFeedInboxFlow 补偿流程。
- P2：补充游标并发语义与 FeedInbox 生命周期接口 DeleteBefore。

## 2026-04-21 分片路由细化修复
- 为避免跨分片批读/批写歧义：MomentRepository 增加 BatchGetByPostIds（按 post_id 由仓储内部路由）。
- FanoutWorkerService 改为先按 friend_id 路由分组，再按 chunk 批量写 FeedInbox。
- FeedInboxRepository 写接口改为 BatchUpsertByRoute，明确批写分片边界。

## 
2026-04-22 00:16:49
 时间线读取链路优化
- 将 page size 上限抽为 MAX_PAGE_SIZE 常量，统一约束。
- FeedInbox->Post 回查前新增 post_id 去重，降低重复批量读取。
- 遇到缺失 Post 时改为跳过单条，避免整页失败。
- 保持从 FeedInbox 读取，不引入读时按 5000 好友动态归并。

## 2026-04-22 高并发治理二次修复（Karpathy最小改动）
- Publish 路径补强：OutboxRepository 明确分区消费契约（ListActivePartitions/CountPendingByPartition/FetchPendingByPartition）与 dead-letter 入口。
- FanoutWorker 补强：引入分区积压软硬阈值、每分区处理配额、author 短窗口合并，降低写风暴和重复好友读取。
- Timeline 补强：请求内 post_id 去重 + 缺失 post 降级，降低随机回表放大导致的整页失败风险。
- Retention 补强：新增 FeedRetentionFlow，硬性约束 365 天保留与每用户 20000 行上限。
- 一致性补强：FriendService 在双边写失败时触发 EnsureEdgePair 修复入口。
- 文档补强：technical-architecture 与 final-delivery-spec 新增高并发强约束验收条款。

## 
2026-04-22 00:41:20
 AGENTS 上下文管理规则更新
- 新增第 21 节：Context 管理与 Skills 触发规则。
- 明确默认基线为 planning-with-files。
- 补充 context-fundamentals / optimization / compression / degradation / filesystem-context 的触发条件与执行顺序。

## 
2026-04-22 00:43:08
 技术文档维护（本轮完善）
- 已将本轮架构改进写入 technical-architecture：时间线读放大治理、链路稳定性增强、路径唯一性约束。
- 已将本轮边界优化写入 technical-architecture：仅伪代码交付、禁止读扩散回退、禁止深分页 offset 默认、禁止全表扫描。
- 已在变更记录补充 2026-04-22 条目，便于后续追踪与面试复盘。

## 2026-04-22 最终评审缺口修复
- 幂等返回缺口：publishMoment 已改为从 IdempotencyRecord.result_ref 返回原始 post_id，重复请求不再返回 0。
- fan-out 批量缺口：FanoutWorker 已改为按最终 FeedInbox item 数切分，避免 friend chunk * event count 造成批量失控。
- Timeline 缺失 post 缺口：TimelineService 已增加 overfetch，遇到缺失 post 时继续消耗后续 inbox 行尽量补足页面。
- 好友修复缺口：EnsureEdgePair 替换为 EnqueueEdgePairRepair，并新增 RepairFriendshipEdgePairFlow 表达持久化补偿。
- 分片路由缺口：IdGenerator 与 MomentRepository 明确 post_id 必须可路由到 Post 分片。
- 交付边界缺口：文档明确 FeedInbox V1 只保证 retention 窗口内的近期时间线。
- 容量证明缺口：文档新增 required_fanout_write_capacity >= publish_qps * avg_friend_count。

## 2026-04-22 V1 全量架构总览文档新增
- 新增 docs/v1-overview.md，系统化汇总当前 V1 的架构、模型、流程、约束映射、边界与验收结论。
- 内容保持现有设计语义，不引入新模块，不扩大范围。

## 2026-04-22 V1 文档最小加分补强
- 在 docs/v1-overview.md 新增“10.4 未来演进（仅说明）”，明确当前统一 push 合理性与未来 push/pull hybrid 演进方向。
- 未改变 V1 主架构与当前实现范围，仅做说明性增强。

## 2026-04-22 上下文压缩快照（skills管理）
- 当前基线：V1 主架构固定为 Friendship + Post + FeedInbox，时间线主路径为 FeedInbox -> BatchGet(Post)。
- 当前状态：已完成最终复审要求的修补与文档沉淀，包含 v1-overview.md 与 v1-enhancement-notes.md。
- 边界锁定：仅 docs/pseudocode，不新增题外功能，不重构主架构。
- 后续操作原则：优先做结构整理、去重、表达压缩；非用户明确要求不做语义级改动。

## 2026-04-22 增强方案实际落盘复核
- 复核结论：增强方案已实际进入文档与最小伪代码，不是纯文字承诺。
- 文档侧已落盘：
  - `docs/v1-enhancement-notes.md`（V1 enhancement notes 骨架）
  - `docs/v1-overview.md`（10.4 未来演进，仅说明）
  - `docs/technical-architecture.md`（最终缺口修复条款）
  - `docs/final-delivery-spec.md`（有条件通过修正项）
- 伪代码侧最小落盘：
  - 幂等重复发布返回原 `post_id`（`result_ref` 语义）
  - fan-out 按最终 FeedInbox item 数分批
  - timeline overfetch + 去重 + 缺失降级补页
  - 好友双边修复改为持久化补偿任务入口
## 2026-04-22 本轮补强新增点
- `docs/v1-enhancement-notes.md` 的“高活跃用户 / 大V 演进说明”已从一句话骨架扩写为工程化 3 段正文。
- `domain/fanout_worker_service.pseudo` 已增加边界注释：V1 固定全量 push，hybrid 仅未来演进，不进入当前执行链路。
- 处理过程发现一次文件编码风险（非 UTF-8），已恢复为 UTF-8 并回读确认内容正确。

## 2026-04-22 friend list cache 增强落盘
- 已扩写 `docs/v1-enhancement-notes.md` 第 5 节，明确 friend list cache 只做可选读优化、`Friendship` 仍是真相源。
- 已在 `domain/fanout_worker_service.pseudo` 加入最小伪代码优化：优先读 `FriendListCacheRepository`，miss/失效回源 `FriendRepository.ListByUser`，并短 TTL 回填。
- 本次不改变 V1 主链路，不引入复杂缓存协议，不修改 `Friendship` 数据模型语义。
- 为保证契约一致性，已确认 `storage/repositories.pseudo` 已包含 `FriendListCacheRepository` 可选接口定义（本轮无需新增）。
## 2026-04-22 可观测性小节增强落盘
- 已扩写 `docs/v1-enhancement-notes.md` 第7节，明确该能力属于工程治理增强而非业务扩展。
- 指标已覆盖并与核心功能映射：publish qps、fan-out lag、inbox write failure rate、timeline read latency(p95/p99)。
- 明确不展开为完整告警平台或监控系统实现，仅保留最小可观测性说明。

## 2026-04-22 第6节扩写完成（fan-out写扩散量化）
- docs/v1-enhancement-notes.md 第6节已扩写为工程化 3 段：约束驱动写放大、容量公式、异步分发取舍。
- 明确保留 V1 主链路不变，说明属于现有 tradeoff 的量化解释而非新设计。
## 2026-04-22 高负载降级伪代码落盘
- 已在 fan-out worker 引入可执行降级分支：`SOFT_DEGRADED` 降速消费、`HARD_DEGRADED` 暂停该分区消费并保留事件。
- 已在 publish flow 与 moment service 明确“发布成功优先、时间线可短暂延迟、最终一致性不变”。
- 已扩写 `docs/v1-enhancement-notes.md` 第8节，强调该策略是高并发工程权衡而非功能缺失。
