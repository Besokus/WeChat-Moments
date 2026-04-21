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
