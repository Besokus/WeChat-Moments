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
